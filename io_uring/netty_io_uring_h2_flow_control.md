# io_uring 异步写如何让 Netty HTTP/2 把大 body 切成 1KB 碎片

note: 本文记录的是 Claude Opus 5.5 与我一起排查 Netty io_uring 在 HTTP/2 场景下 CPU 异常的过程。文中 Netty 代码均基于 [`4.2@006a2f1d92`](https://github.com/netty/netty/tree/006a2f1d92db733d10112de251600da7b6f8ec51)。

## TL;DR

- 把 Netty 的 HTTP/2 服务从 NIO（epoll）切到 io_uring，event loop 的 CPU 反而**虚高**了。问题不在 io_uring 慢，而在于**它的写是异步完成的**。
- NIO 是同步写，flush 时字节立刻离开 `ChannelOutboundBuffer`，写水位马上回落。io_uring 要等 CQE 到来才 `removeBytes`，在这之前水位一直被占着。
- h2 的 flow controller 按 channel 剩余的可写字节分配写入预算。在途字节一旦越过高水位（默认 64KB），后续响应就会积压在 flow controller 里。积压解除时，多个 stream 同时待发，默认的 `WeightedFairQueueByteDistributor` 以 `allocationQuantum = 1KB` 轮转，把大 body 切成一堆 1KB 的 DATA frame。ByteBuf、slice、promise 的数量膨胀近 10 倍，zero-copy 的阈值也永远达不到。
- 即使是 `HttpToHttp2ConnectionHandler` + 每个响应各自 `writeAndFlush`，在 io_uring 下也会中招；在 NIO 下不会。
- 参考 grpc-java，改用 `UniformStreamByteDistributor` + `minAllocationChunk = 16KB` 后，io_uring 的 event loop CPU 比 WFQ 默认下降 33% 到 63%，延迟也更低。HTTP/2 的优先级权重已在 RFC 9113 中被废弃，1KB 的轮转粒度在内网带宽下也换不来可感知的公平性。Vert.x 5 同样默认改用了 Uniform。
- **核心教训：io_uring 的异步写改变了"一次写什么时候算完成"的语义，所有隐含"写是同步完成"这个假设的上层机制（写水位、流控预算），都可能出现意料之外的行为。**

## 1. 问题：命中不了的 SENDMSG_ZC，和比 NIO 还高的 CPU

我在研究 Netty io_uring 的 `SENDMSG_ZC` 在 h2 上的优化时，碰到一件很奇怪的事：向同一个连接的多个 stream 写入远超 zero-copy 阈值的大 body，后续的 DATA frame 竟然**没有一个**达到 `SENDMSG_ZC` 的阈值。用 bpftrace 统计 `io_uring:io_uring_submit_req`，16KB body、5000 rps 下，每个请求的 `SEND_ZC`/`SENDMSG_ZC` 都是 0，只有零星的 `WRITEV`（每个请求 0.36 次，即一次 writev 带走约 3 个响应）。

退回到普通 writev 后，更怪的事情来了：**io_uring 的 event loop CPU 比 NIO 实现还高**。换成线上的写法（`HttpToHttp2ConnectionHandler`，每个响应各自 `writeAndFlush(FullHttpResponse)`），64KB body、每个连接并发 4 个请求时，NIO 每个请求写出 12 个 ByteBuf、event loop 耗时 11.27 µs；io_uring 每个请求写出 43.7 个 ByteBuf、耗时 17.36 µs，多了 54%（详见第 3 节）。同样的服务换成 HTTP/1.1 的 `FullHttpResponse`，两个 transport 基本没有差别（云上 64KB 时是 15.5 对 15.7 µs），差距只出现在 h2 上。

排查下来，罪魁祸首是 `WeightedFairQueueByteDistributor`：它把大 body 切成了大量 1KB 的 DATA frame。为什么只有 io_uring 会这样？

## 2. 原因：写水位的变化方式不同

### 2.1 h2 的写入预算跟着 channel 的可写状态走

h2 写 DATA 分两步。第一步，`DefaultHttp2RemoteFlowController.writePendingBytes()` 算出这一轮能写多少，再交给 `StreamByteDistributor` 分给各个 stream（[`DefaultHttp2RemoteFlowController.java#L611-L634`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/DefaultHttp2RemoteFlowController.java#L611-L634)）：

```java
int bytesToWrite = writableBytes();
for (;;) {
    if (!streamByteDistributor.distribute(bytesToWrite, this) ||
        (bytesToWrite = writableBytes()) <= 0 ||
        !isChannelWritable0()) {
        break;
    }
}
```

预算取决于 channel 还能写多少字节（[`#L237-L262`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/DefaultHttp2RemoteFlowController.java#L237-L262)）：

```java
private int maxUsableChannelBytes() {
    // If the channel isWritable, allow at least minUsableChannelBytes.
    int channelWritableBytes = (int) min(Integer.MAX_VALUE, ctx.channel().bytesBeforeUnwritable());
    int usableBytes = channelWritableBytes > 0 ? max(channelWritableBytes, minUsableChannelBytes()) : 0;
    return min(connectionState.windowSize(), usableBytes);
}
```

`bytesBeforeUnwritable()` 是写高水位减去 `ChannelOutboundBuffer` 中**还没写完**的字节（[`ChannelOutboundBuffer.java#L787`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/transport/src/main/java/io/netty/channel/ChannelOutboundBuffer.java#L787)）。channel 不可写时预算是 0，数据只能留在 flow controller 里排队。

第二步，`FlowControlledData.write()` 只写出本轮分到的 `allowedBytes`（[`DefaultHttp2ConnectionEncoder.java#L526`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/DefaultHttp2ConnectionEncoder.java#L526)），然后由 `DefaultHttp2FrameWriter.writeData()` 按 `maxFrameSize` 切帧。每个帧都会单独分配一个 9 字节的帧头 buffer，再加上一个 payload 的 retained slice（[`DefaultHttp2FrameWriter.java#L162-L183`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/DefaultHttp2FrameWriter.java#L162-L183)）。

所以**实际 DATA frame 的大小 = min(本轮分到的字节数, maxFrameSize)**。`SETTINGS_MAX_FRAME_SIZE`（默认 16KB）只是上限，多个 stream 并发时，真正起作用的是 distributor 每轮分出去的字节数。

### 2.2 WFQ 默认 1KB 轮转

`WeightedFairQueueByteDistributor` 的默认 quantum 是 1KB（[`Http2CodecUtil.java#L119`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/Http2CodecUtil.java#L119)、[`WeightedFairQueueByteDistributor.java#L90`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/WeightedFairQueueByteDistributor.java#L90)）。多个 stream 同时待发时，每个 stream 每轮分到的字节数是（[`#L313-L339`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/WeightedFairQueueByteDistributor.java#L313-L339)）：

```java
int nsent = distribute(nextChildState == null ? maxBytes :
                min(maxBytes, (int) min((nextChildState.pseudoTimeToWrite - childState.pseudoTimeToWrite) *
                                   childState.weight / oldTotalQueuedWeights + allocationQuantum, MAX_VALUE)),
                writer, childState);
```

几个 stream 刚开始时伪时间相同，所以每轮只能分到约 `allocationQuantum` = 1KB，然后就切到下一个 stream。**只要 flush 时有两个以上的 stream 同时待发，大 body 就会被切成一串约 1KB 的 DATA frame**，交错写出：`[9B][~1KB][9B][~1KB]...`。只有一个 stream 待发时，它能拿到全部预算，不会被切成 1KB。

关键问题因此变成：**什么时候会有多个 stream 同时待发？**

### 2.3 NIO：同步写，水位立刻回落

NIO 的 `doWrite()` 在 flush 里直接 `writev`，写多少就立刻 `removeBytes` 多少（[`NioSocketChannel.java#L423-L431`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/transport/src/main/java/io/netty/channel/socket/nio/NioSocketChannel.java#L423-L431)）：

```java
final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
...
in.removeBytes(localWrittenBytes);
```

内网场景下 socket 发送缓冲区几乎总有空间，所以一个大 DATA frame flush 完，outbound buffer 就清空了，水位马上回落。下一个 stream 写入时看到的预算是充足的，flow controller 里只有它自己，于是整个 body 以满帧（16KB）发出。

### 2.4 io_uring：异步写，水位要等 CQE

io_uring 的 flush 只是提交一个 `WRITEV` SQE（[`AbstractIoUringStreamChannel.java#L265-L291`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/transport-classes-io_uring/src/main/java/io/netty/channel/uring/AbstractIoUringStreamChannel.java#L265-L291)），字节要等**下一轮 event loop 处理 CQE 时**才移除（[`#L690-L720`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/transport-classes-io_uring/src/main/java/io/netty/channel/uring/AbstractIoUringStreamChannel.java#L690-L720)）：

```java
boolean writeComplete0(byte op, int res, int flags, long data, int outstanding) {
    ...
    if (res >= 0) {
        channelOutboundBuffer.removeBytes(res);
    }
    ...
}
```

而且每个 channel 同一时刻只有一个写在飞，在途期间到来的 flush 只能往 outbound buffer 里追加。于是：

1. 同一个 read loop 里处理了同一连接上的多个请求，每个响应 `writeAndFlush` 后，字节都还留在 outbound buffer 里，水位只升不降。
2. 积累几个响应后就越过了 64KB 高水位（`ChannelOutboundBuffer.incrementPendingOutboundBytes` → `setUnwritable`，[`#L198-L205`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/transport/src/main/java/io/netty/channel/ChannelOutboundBuffer.java#L198-L205)），channel 变成不可写，预算变成 0，后续响应只能积压在 flow controller 里。
3. CQE 回来后 `removeBytes`，水位降到低水位以下，触发 `channelWritabilityChanged`，进而 `Http2ConnectionHandler.flush()` 和 `writePendingBytes()`（[`Http2ConnectionHandler.java#L195-L207`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/Http2ConnectionHandler.java#L195-L207)）。这时 flow controller 里**同时有多个 stream 待发**，WFQ 开始按 1KB 轮转。

**一句话总结：NIO 和 io_uring 的写水位曲线不一样。NIO 每写完一次就回落，WFQ 永远只看到一个 stream；io_uring 的水位要等 CQE 才回落，积压让 WFQ 进入多 stream 轮转，把大 body 切成了 1KB 的碎片。**

### 2.5 `HttpToHttp2ConnectionHandler` 也逃不掉，`Http2MultiplexHandler` 更糟

我们线上用的是 `HttpToHttp2ConnectionHandler`：每个 `FullHttpResponse` 直接 `writeAndFlush`，整个 body 交给 encoder（[`HttpToHttp2ConnectionHandler.java#L135`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/HttpToHttp2ConnectionHandler.java#L135)）。在 NIO 下，每个响应 flush 时只有自己一个 stream 待发，确实不会交错；但在 io_uring 下，一旦积压，同样会触发上面的流程。

`Http2MultiplexHandler` 的情况更糟：子 channel 的 flush 在父 channel 的 read loop 期间是空操作（[`AbstractHttp2StreamChannel.java#L1209-L1222`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/AbstractHttp2StreamChannel.java#L1209-L1222)），要等到 read complete 时统一 flush 一次（[`Http2MultiplexHandler.java#L391-L406`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/Http2MultiplexHandler.java#L391-L406)）：

```java
public void flush() {
    // If we are currently in the parent channel's read loop we should just ignore the flush.
    // We will ensure we trigger ctx.flush() after we processed all Channels later on and
    // so aggregate the flushes.
    if (!writeDoneAndNoFlush || isParentReadInProgress()) {
        return;
    }
    ...
}
```

所以同一批请求的响应天然就会同时待发，**即使在 NIO 下也会被 WFQ 切碎**，在子 channel 里调用 `writeAndFlush` 也没用。第 1 节中 16KB 一次 zero-copy 都没命中，就是这个原因（压测工具 h2load 的 `--rps` 每 10ms 成批发请求，会让同一连接上的多个请求落进同一次读）。

### 2.6 除了切碎，异步完成本身也有代价

即使没有多个 stream 同时待发，io_uring 在 h2 上也比 NIO 贵。云上两台 4 vCPU 虚拟机走内网压测，h2c，`Http2MultiplexHandler`，固定速率，event loop 线程每个请求的 CPU：

| body | NIO | io_uring（未开 zero-copy） |
|---|---|---|
| 16KB @ 5000 rps | 14.4 µs | 16.9 µs |
| 64KB @ 1500 rps | 28.3 µs | 32.4 µs |
| 256KB @ 400 rps | 79.1 µs | 85.6 µs |

其中 64KB @ 1500 rps 时，每个连接平均约 10ms 才来一个请求，几乎不会出现多个 stream 同时待发。这时的差距分成两部分。第一部分是多一轮 writev：一个 64KB 的 h2 响应（数据 + 帧头 + HEADERS）刚好超过 64KB 高水位，NIO 能在同一次 flush 里写完，io_uring 要等 CQE 回来释放预算后，在下一轮 event loop 再提交一次。把高水位调到 1MB 就能消除：

| 配置（64KB @ 1500 rps） | event loop µs/req | 每个请求的 WRITEV | 每个请求的发包次数 |
|---|---|---|---|
| NIO | 25.7 | — | 1.96 |
| io_uring，默认 64KB 高水位 | 31.1 | 1.92 | 2.58 |
| io_uring，1MB 高水位 | 29.1 | 0.96 | 1.95 |

第二部分是剩下的约 3.4 µs。用 `perf stat -t` 只统计 event loop 线程（20 秒，约 1490 rps，两轮）：**两边的指令数完全相同**（2.58/2.53 G 对 2.53/2.56 G），但 io_uring 的周期数多 8% 到 10%，**cache miss 多 35%**（33.9/32.5 M 对 44.8/45.2 M），IPC 从 0.87 降到 0.80。我的推测是，NIO 在 flush 里同步完成写入，buffer、promise、outbound entry 在被访问时都还在缓存里；io_uring 要到下一轮处理 CQE 时才回头释放 buffer、完成 promise、清理 entry，这时这些数据已经被其他工作挤出了缓存。

## 3. 测试

### 3.1 环境

- **本地**：i5-13600KF，loopback。服务端 2 个 event loop，绑在 P 核 0 和 2；压测端绑在 P 核 4/6/8/10。
- **云上**：两台腾讯云 SA9.LARGE8（4 vCPU，内核 7.0，virtio_net，iommu=pt），走内网，带宽被限在约 2.25 Gbps，所以统一用固定速率比较每个请求的 CPU。注意：主机安全 agent 在网卡上挂的 `ETH_P_ALL` 抓包 socket 会让所有 zero-copy 悄悄退化成 copy（发送路径上 `dev_queue_xmit_nit` 会调用 `skb_copy_ubufs`），测试前必须停掉。
- 基线代码：Netty [`4.2@006a2f1d92`](https://github.com/netty/netty/tree/006a2f1d92db733d10112de251600da7b6f8ec51)，JDK 25。

### 3.2 服务端：模拟线上的 `HttpToHttp2ConnectionHandler`

请求经 `InboundHttp2ToHttpAdapter` 聚合成 `FullHttpRequest`，handler 直接 `writeAndFlush(FullHttpResponse)`。body 在启动时构造一次，每次写出 `retainedDuplicate()`，相当于一份缓存好的内容：

```java
DefaultHttp2Connection connection = new DefaultHttp2Connection(true);
installDistributor(connection, distributor, quantum);
pipeline.addLast(new HttpToHttp2ConnectionHandlerBuilder()
                .connection(connection)
                .initialSettings(Http2Settings.defaultSettings().maxConcurrentStreams(1024))
                .frameListener(new InboundHttp2ToHttpAdapterBuilder(connection)
                        .maxContentLength(1 << 20)
                        .propagateSettings(false)
                        .validateHttpHeaders(false)
                        .build())
                .build(),
        new AdaptedH2Handler(body));

// AdaptedH2Handler
protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest req) {
    DefaultFullHttpResponse res = new DefaultFullHttpResponse(HTTP_1_1, OK, body.retainedDuplicate());
    res.headers().setInt(CONTENT_LENGTH, body.readableBytes());
    res.headers().set(HttpConversionUtil.ExtensionHeaderNames.STREAM_ID.text(),
                      req.headers().get(HttpConversionUtil.ExtensionHeaderNames.STREAM_ID.text()));
    ctx.writeAndFlush(res, ctx.voidPromise());
}
```

切换 distributor 的方式：

```java
private static void installDistributor(DefaultHttp2Connection connection, String distributor, int quantum) {
    StreamByteDistributor d;
    if ("uniform".equals(distributor)) {
        UniformStreamByteDistributor u = new UniformStreamByteDistributor(connection);
        if (quantum > 0) {
            u.minAllocationChunk(quantum);
        }
        d = u;
    } else {
        WeightedFairQueueByteDistributor w = new WeightedFairQueueByteDistributor(connection);
        if (quantum > 0) {
            w.allocationQuantum(quantum);
        }
        d = w;
    }
    connection.remote().flowController(new DefaultHttp2RemoteFlowController(connection, d));
}
```

为了看清 transport 实际收到了什么，pipeline 最前面挂了一个出站 handler，统计每个写出的 `ByteBuf` 的大小分布。

### 3.3 压测端：同一连接上成批并发

压测端是一个基于 Netty 的 h2c 客户端：16 个连接，每个连接每 10ms **一次性**打开 N 个 stream（开环），模拟浏览器或 gRPC 在一个连接上的并发请求。每个 stream 记录完成时间；测量窗口前后各请求一次 `/stats`，拿到服务端 event loop 线程的累计 CPU（`ThreadMXBean.getThreadCpuTime`）和 buffer 大小直方图，相减得到严格对齐测量窗口的数据：

```java
loop.scheduleAtFixedRate(() -> {
    for (int k = 0; k < burst; k++) {
        request(ch, reqHeaders, rec);   // Http2StreamChannelBootstrap(ch).open() -> HEADERS(endStream)
    }
}, initialDelayMicros, interval * 1000L, TimeUnit.MICROSECONDS);
```

### 3.4 结果一：NIO 不切，io_uring 切

`HttpToHttp2ConnectionHandler` + `writeAndFlush`，WFQ（Netty 默认），默认 64KB 高水位：

| body | 每个连接的并发 | transport | 每个请求的 ByteBuf 数 | 其中 1-2KB 小块 | event loop µs/req |
|---|---|---|---|---|---|
| 16KB | 4 | NIO | 4.0 | 0 | 8.22 |
| 16KB | 4 | io_uring | 4.0 | 0 | 8.09 |
| 16KB | 16 | NIO | 4.0 | 0 | 6.02 |
| 16KB | 16 | **io_uring** | **16.4** | 2.5 | **7.93（+32%）** |
| 32KB | 4 | NIO | 6.0 | 0 | 8.07 |
| 32KB | 4 | **io_uring** | **8.4** | 1.3 | 8.23 |
| 64KB | 4 | NIO | 12.0 | 0 | 11.27 |
| 64KB | 4 | **io_uring** | **43.7** | 16.2 | **17.36（+54%）** |
| 64KB | 4 | io_uring，1MB 高水位 | 10.0 | 0 | 10.21 |

NIO 在所有组合下都保持满帧。io_uring 只要在途字节越过高水位（body 越大、并发越高越容易发生）就开始切分；把高水位调到 1MB 后切分消失。

### 3.5 结果二：换 distributor

io_uring，`HttpToHttp2ConnectionHandler` + `writeAndFlush`，**默认 64KB 高水位**：

| body | 并发 | distributor | 每个请求的 ByteBuf 数 | 16KB 帧 | event loop µs/req | 完成时间 p50 |
|---|---|---|---|---|---|---|
| 64KB | 4 | WFQ（Netty 默认） | 43.8 | 1.9 | 18.0 | 58 µs |
| 64KB | 4 | Uniform（Vert.x 5 默认） | 13.9 | 3.8 | 12.0（−33%） | 40 µs |
| 64KB | 4 | Uniform + 16KB（grpc-java） | 13.9 | 3.7 | 12.1 | 40 µs |
| 64KB | 4 | WFQ + quantum 16KB | 12.5 | 4.0 | 11.2 | 39 µs |
| 64KB | 16 | WFQ（Netty 默认） | 109.5 | 0.5 | 28.9 | 423 µs |
| 64KB | 16 | Uniform（Vert.x 5 默认） | 49.5 | 0.6 | 17.0（−41%） | 240 µs |
| 64KB | 16 | **Uniform + 16KB（grpc-java）** | **12.6** | 4.0 | **10.6（−63%）** | **144 µs** |
| 16KB | 16 | WFQ（Netty 默认） | 16.4 | 0.5 | 7.93 | 62 µs |
| 16KB | 16 | **Uniform + 16KB** | **4.3** | 1.0 | **5.25（−34%）** | 49 µs |

`UniformStreamByteDistributor` 每次给每个 stream 分 `max(minAllocationChunk, 预算 / 待发 stream 数)`（[`UniformStreamByteDistributor.java#L96`](https://github.com/netty/netty/blob/006a2f1d92db733d10112de251600da7b6f8ec51/codec-http2/src/main/java/io/netty/handler/codec/http2/UniformStreamByteDistributor.java#L96)）：

```java
final int chunkSize = max(minAllocationChunk, maxBytes / size);
```

- 并发 4 时，预算平分后每个 stream 有 8KB 到 16KB，Uniform 默认配置就基本不切了。
- 并发 16 时，平分后只剩 2KB 到 4KB，Uniform 默认配置又开始退化（每个请求 49.5 个 ByteBuf）。
- `minAllocationChunk = 16KB` 让每次分配至少一个满帧，在所有并发度下都不切分。16KB 并发 16 时，io_uring 的 event loop CPU（5.25 µs）甚至低于 NIO（6.02 µs）。

更粗的粒度没有损害延迟，反而因为 CPU 开销下降，所有 stream 都更快完成了。

## 4. 别人也踩过：grpc-java 和 Vert.x

**grpc-java** 改了两次：

- 2017 年，commit [`7b821a0e50` "netty: increase message quantum"](https://github.com/grpc/grpc-java/commit/7b821a0e50a4059cd22dc63f81134fcfbd9c7c4f) 把 WFQ 的 `allocationQuantum` 设成 16KB，注释是 `// Make benchmarks fast again.`。commit message 里写道："The size was ~1024bytes previously, which resulted in the `writev` syscalls sending many smaller chunks before hitting the low water mark……When sending as many 1MB messages as possible it nearly doubles the rate."
- 2025 年，PR [#11954 "netty: Swap to UniformStreamByteDistributor"](https://github.com/grpc/grpc-java/pull/11954) 改用 Uniform，`minAllocationChunk = 16 * 1024`（[`AbstractNettyHandler.java#L53`](https://github.com/grpc/grpc-java/blob/1084e9707d05a64dcb16a934bc03f7e2f68d013e/netty/src/main/java/io/grpc/netty/AbstractNettyHandler.java#L53)、[`NettyServerHandler.java#L261-L265`](https://github.com/grpc/grpc-java/blob/1084e9707d05a64dcb16a934bc03f7e2f68d013e/netty/src/main/java/io/grpc/netty/NettyServerHandler.java#L261-L265)）。起因是 issue [#11659](https://github.com/grpc/grpc-java/issues/11659)：WFQ 会让同一线程按顺序发出的 RPC 在服务端乱序到达。grpc-java 也在 Netty issue [#14918](https://github.com/netty/netty/issues/14918) 中提到，因为优先级已被废弃，想换成 Uniform。

**Vert.x**：

- 4.x，PR [#5071 "HTTP/2 configurable stream byte distributor"](https://github.com/eclipse-vertx/vert.x/pull/5071)：可以通过系统属性 `vertx.h2StreamByteDistributor=uniform` 切换到 Uniform，默认仍是 WFQ。
- 5.x，PR [#5077](https://github.com/eclipse-vertx/vert.x/pull/5077)：默认改为 Uniform，理由是 "does not use stream priorities and yields better performance since it consumes less CPU"（另见[迁移指南](https://vertx.io/docs/guides/vertx-5-migration-guide/#_use_of_uniform_byte_distributor_for_http2_streams)）。但 Vert.x 没有调 chunk 大小，用的仍是默认的 1KB（[`VertxHttp2ConnectionHandlerBuilder.java#L155-L160`](https://github.com/eclipse-vertx/vert.x/blob/c206d7a70f942bf1e191e2dcc89118cb72edfecd/vertx-core/src/main/java/io/vertx/core/http/impl/http2/codec/VertxHttp2ConnectionHandlerBuilder.java#L155-L160)），所以高并发积压时还会出现上表中"Uniform 默认、并发 16"那一行的退化。

## 5. 结论

### 5.1 优先级已废除，1KB 的粒度也没有意义了

**"加权"已经没有输入。** [RFC 9113 §5.3](https://www.rfc-editor.org/rfc/rfc9113.html#section-5.3) 废弃了 RFC 7540 的优先级方案，新的客户端不再发送依赖关系和权重，所有 stream 权重相同。WFQ 的 CFS 式伪时间计算退化成了带额外开销的轮转，还会造成 grpc-java #11659 那种乱序。

**轮转粒度还有意义，但 1KB 太小。** 它控制的是并发时每个 stream 一次能连续写多少，本质是首字节公平性和帧数/CPU 之间的取舍。按 12 Gbps 内网粗算：

| 粒度 | 线上发送时间 | 每多一个竞争 stream，最多多等 |
|---|---|---|
| 1KB | 约 0.7 µs | 0.7 µs |
| 16KB | 约 11 µs | 11 µs |
| 64KB | 约 44 µs | 44 µs |

- 内网 RTT 在几十到几百微秒，单个请求在 event loop 上的处理时间也有几十微秒，16KB 粒度带来的约 11 µs 等待根本测不出来；1KB 换来的那点公平性被完全淹没，代价（约 10 倍的帧数和明显的 CPU 开销）却是实实在在的。
- 更关键的是，Netty 的调度只决定字节**进入 socket 的顺序**。数据一旦进入内核发送缓冲区（自动调整后可达数 MB），就是 FIFO 发出。Netty 每次只调度高水位范围内的几十 KB，再精细的轮转，在数 MB 的内核队列面前也没有意义。
- 只有在低带宽链路上（比如面向公网的移动端，10 Mbps 下 16KB 要约 13 ms），小粒度才可能对首字节时间有一点帮助，而且也会被内核发送缓冲区稀释。

### 5.2 我们的做法

1. **改用 `UniformStreamByteDistributor` + `minAllocationChunk(16 * 1024)`**，和 grpc-java 一致：

   ```java
   DefaultHttp2Connection connection = new DefaultHttp2Connection(true);
   UniformStreamByteDistributor distributor = new UniformStreamByteDistributor(connection);
   distributor.minAllocationChunk(16 * 1024);
   connection.remote().flowController(new DefaultHttp2RemoteFlowController(connection, distributor));
   HttpToHttp2ConnectionHandler handler = new HttpToHttp2ConnectionHandlerBuilder()
           .connection(connection)
           .frameListener(...)
           .build();
   ```

   我们的带宽大到几乎打不满，优先级也已经被废弃，所以这个修改对我们是纯收益。

2. **用 io_uring 时，同时调大 `WRITE_BUFFER_WATER_MARK`**（比如高水位 1MB），从源头上减少"异步完成导致预算被冻结、多个 stream 积压"的情况。

3. **可以给 Netty 提的改进**：h2 默认使用 Uniform，且 chunk 至少等于 `SETTINGS_MAX_FRAME_SIZE`；io_uring transport 在计算可写字节时考虑"已提交、未完成"的写，或者让 h2 的 flush 在 CQE 回来后继续，避免异步写把写入预算冻结一整轮。

### 5.3 教训

io_uring 的异步写不只是"更快的系统调用"。**它改变了"一次写什么时候算完成"的语义。** Netty 的写水位、h2 的流控预算，以及流控之上的 stream 调度，都隐含了"flush 返回时字节已经交给内核"的假设。NIO 满足这个假设，io_uring 不满足。从 epoll 迁移到 io_uring 时，除了 transport 本身，还要回头检查这些建立在同步写语义上的上层机制。
