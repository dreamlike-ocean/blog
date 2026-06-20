# Netty io_uring linked SQE 设计方案 和 cancel 行为调研

note: 本文记录的是 gpt5.5 xhigh模型与我一起 为 Netty `io_uring` 适配Linked SQE 过程中的资料发现和思考整理

## Netty 为什么不能直接支持 linked SQE shared user_data

核心问题不是 CQE 回来后触发正确的 callback，

而是提交侧必须保证一批 linked SQE 不会被 `enqueueSqe()` 中途切成两次 submit。

Netty 的 `SubmissionQueue.enqueueSqe()` 是单条 SQE 的工具方法。

它发现 SQ 满了以后，会先 `submit()` 当前队列，再继续写下一条 SQE：

```java
        int pending = tail - head;
        if (pending == ringEntries) {
            int submitted = submit();
            //...
        }
int sqe = sqeIndex(tail++, ringMask);
         // 省略填充
```

这对普通 SQE 很自然，因为每个 SQE 本来就是独立提交单元。

但是 linked SQE 不是。

linked batch 的语义来自相邻 SQE 上的 `IOSQE_IO_LINK`， 也就是这一条和下一条组成一个 chain。

如果 Netty 不先检查 SQ 剩余空间，而是逐条 enqueue，就可能出现下面这种形态：

```text
SQ 剩余 2 个位置，linked batch 有 3 个 SQE

enqueue linked[0]   // 带 IOSQE_IO_LINK
enqueue linked[1]   // 带 IOSQE_IO_LINK
// SQ 满，enqueueSqe() 内部触发 submit()
enqueue linked[2]   // 最后一条，不带 IOSQE_IO_LINK
```

对 Netty 的实现来说，这已经不是一个完整预留过的 linked batch 了。 前两条和第三条跨过了 `io_uring_enter()` 边界，

所以 linked path 不能直接复用普通逐条 enqueue 的行为，而是要先为整批 SQE 做容量探测：

```java
private void submitLinked0(IoUringLinkedIoOps linkedOps, long token) {
    SubmissionQueue submissionQueue = ringBuffer.ioUringSubmissionQueue();
    submissionQueue.ensureWritable(linkedOps.size());
    for (int i = 0; i < linkedOps.size(); i++) {
        IoUringIoOps ioOps = linkedOps.op(i);
        pendingOps.registerNormal(token, id, ioOps.opcode(), ioOps.userData());
        submissionQueue.enqueueSqe(..., token, ...);
    }
    outstandingCompletions += linkedOps.size();
}
```

`ensureWritable()` 做两件事：

```java
void ensureWritable(int sqeCount) {
    checkClosed();
    if (sqeCount > ringEntries) {
        throw new IllegalArgumentException(...);
    }
    if (remaining() < sqeCount) {
        submit();
    }
    if (remaining() < sqeCount) {
        throw new RuntimeException(...);
    }
}
```

第一，linked batch 本身不能比 SQ ring 还大。

第二，如果当前 SQ 剩余位置不够，先 submit 一次腾空间。

做完这个检查以后，后面的 `enqueueSqe()` 在这个 batch 内就不会因为 SQ 满而偷偷 submit。

## long user_data 的 slow path

Netty 本来就已经支持 long `userData`，只是它不一定直接放进 kernel 的 `cqe.user_data`。

fast path 会把 registration id、opcode 和短 `userData` pack 到一个 long 里。

这样 CQE 回来时直接解码，Netty 不需要额外对象或表项去维护这条 SQE 的状态， netty内部路径上也就没有额外内存分配。

```java
// fast path的udata编码方式
//  63             32 31      24 23      16 15       0
// [     id        ][ unused ][   op    ][   data    ]
long userData = ioOps.userData();

if (canUseFastPath(userData)) {
    long packedSeq = UserData.encode(id, ioOps.opcode(), (short) userData);
    submitFastPath0(ioOps, packedSeq);
    return packedSeq;
}
```

long 值塞不进 fast path 时，就走 slow path。

真实源码里生成 slow-path token 的方法叫 `PendingOpMap.nextToken()`：

```java
long nextToken() {
    long sequence = nextSequence.getAndIncrement();
    if (sequence <= 0) {
        throw new IllegalStateException("slow path sequence overflow");
    }
    return token(sequence);
}
```

这个 token 是负数，巧妙地利用 registration id 限制为正数这一点 和 packed fast-path `user_data` 区分。

提交侧把 token 写进 SQE，把真实 long `userData` 存到 `PendingOpMap`：

```java
private void submitSlowPath0(IoUringIoOps ioOps, long token, long userData) {
    pendingOps.registerNormal(token, id, ioOps.opcode(), userData);
    submissionQueue.enqueueSqe(..., token, ...);
}
```

completion 侧按符号位分流：

```java
if (udata >= 0) {
    handleFastPath(..., udata, ...);
} else {
    handleSlowPath(..., udata, ...);
}
```

slow path 再拿 token 去 `PendingOpMap` 找回原始 long `userData` 和 `op` 等元数据：

```java
int slot = pendingOps.findSlot(udata);
long userData = pendingOps.userData(slot);

if ((flags & Native.IORING_CQE_F_MORE) == 0) {
    pendingOps.release(slot);
}
```

## linked SQE 为什么使用共享 user_data 的设计 即每一个的SQE user_data 保持一致

Netty API 的签名是 `long IoRegistration.submit(IoOps)`， 一次 submit 返回一个 long id，linked batch 虽然里面有多个 SQE，但是 API 没有地方返回多个 id。

所以 linked batch 的现实选择是：这一批 SQE 共享一个 Netty slow-path token。

每个 SQE 写给 kernel 的 `user_data` 都是同一个 token，真正属于每个 SQE 的原始 `userData` 留在 Java 侧。

这听起来像是在 map 里放重复 key， 但 `PendingOpMap` 不是一个会按 key 去重的 `Map`。 它是开放寻址表，而且 `registerNormal()` 没有“key 已存在就替换”的逻辑。 

```java
long existing = tokens[index];
if (existing == EMPTY) {
    insertAt(..., token, registrationId, op, userData);
    return;
}
```

普通 slow path 下，`nextSequence` 单调递增，所以每次 `nextToken()` 默认生成不同 token。 这个表看起来像 map，是因为普通路径不会放重复 key。

所以linked path 可以利用这个特性：`PendingOpMap` 不会去重，同一个 token 连续插入时，会沿着开放寻址的探测链占用不同槽位。

再叠加 linked SQE 的 FIFO completion，提交顺序和消费顺序刚好能对上：

```java
long token = pendingOps.nextToken();

for (int i = 0; i < linkedOps.size(); i++) {
    IoUringIoOps ioOps = linkedOps.op(i);
    pendingOps.registerNormal(token, id, ioOps.opcode(), ioOps.userData());
    submissionQueue.enqueueSqe(..., token, ...);
}
```

completion 回来时，`findSlot(token)` 找到当前最靠前的 live entry；`release()` 以后，下一条同 token entry 暴露出来。

这里用的不是额外的顺序表，而是 linked SQE 的 FIFO 语义和开放寻址探测链的自然顺序。

- io_uring linked SQE 提供 FIFO completion 语义，

- `PendingOpMap` 是开放寻址表而不是会去重的 `Map`，

- Netty linked path 又先保证整批 SQE 不被中途 submit 切开。

三件事合在一起，重复 token 反而正好符合这个模型。

## linked timeout 有什么用以及如何实现的

linked SQE 解决的是顺序和失败传播，但它本身不表达 deadline。

`IORING_OP_LINK_TIMEOUT` 填补的是 linked SQE 的 deadline 语义缺陷：

给 linked chain 里的前一个 request 挂一个 timeout, 前驱 request 在 deadline 前完成，内核会摘掉相邻的 link timeout， 并让 timeout SQE 以 `-ECANCELED` 完成；

timeout 先到，内核会尝试 cancel 这个前驱 request, 它不是“整条 chain 的总超时”, `LINK_TIMEOUT` 只约束它前面的那个 linked request； 

如果一个组合操作有多个阶段都需要 deadline，就要在对应阶段后面放对应的 `LINK_TIMEOUT`，或者明确只约束其中一个关键阶段。

注意 liburing 里普通 timeout 和 linked timeout 的 SQE 作用不太一样

普通 `IORING_OP_TIMEOUT` 是一个独立请求，`count` 会放进 `sqe->off`, 如果 `count == 0`，它就是纯时间 timeout, 如果 `count > 0`，它还可以被 CQE 数量推进 也就是它是约束io_uring_enter的阻塞条件的

linked timeout 不支持这个 `count` 语义， 内核 prep 时看到 `off && is_timeout_link` 会直接返回 `-EINVAL`，他的时间参数来自 `struct __kernel_timespec`。 默认是相对时间；

带 `IORING_TIMEOUT_ABS` 后就是绝对时间，clock 默认是 `CLOCK_MONOTONIC`，也可以通过 `IORING_TIMEOUT_BOOTTIME` 或 `IORING_TIMEOUT_REALTIME` 改成 `CLOCK_BOOTTIME` / `CLOCK_REALTIME`：

```c
return (flags & IORING_TIMEOUT_ABS) ? HRTIMER_MODE_ABS
                                    : HRTIMER_MODE_REL;
```

如果是绝对时间，内核还会按 clock 做 time namespace 转换：

```c
if (flags & IORING_TIMEOUT_ABS)
    *time = timens_ktime_to_host(io_flags_to_clock(flags), *time);
```

源码位置：
[Linux `io_parse_user_time()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/timeout.c#L55-L75)、
[Linux `io_translate_timeout_mode()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/timeout.c#L523-L527)

linked timeout 的关键点在 prep 阶段。它必须前面已经有 linked request，否则 `!link->head` 直接 `-EINVAL`。内核把 timeout 挂到当前 link chain 的最后一个 request 上，并给这个前驱打上 `REQ_F_ARM_LTIMEOUT`：

```c
timeout->head = link->last;
link->last->flags |= REQ_F_ARM_LTIMEOUT;
hrtimer_setup(&data->timer, io_link_timeout_fn,
              io_timeout_get_clock(data), data->mode);
```

源码位置：
[Linux `__io_timeout_prep()` linked timeout branch](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/timeout.c#L616-L629)

前驱 request 真正 issue 时，`__io_issue_sqe()` 会看到 `REQ_F_ARM_LTIMEOUT`，先把 linked timeout request 准备好；前驱 issue 完以后再 `io_queue_linked_timeout()` 启动 hrtimer，并把 timeout 放进 `ctx->ltimeout_list`：

```c
if (req->flags & REQ_F_ARM_LTIMEOUT)
    link = __io_prep_linked_timeout(req);

ret = def->issue(req, issue_flags);

if (link)
    io_queue_linked_timeout(link);
```

```c
hrtimer_start(&data->timer, data->time, data->mode);
list_add_tail(&timeout->list, &ctx->ltimeout_list);
```

源码位置：
[Linux `__io_issue_sqe()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/io_uring.c#L1377-L1405)、
[Linux `io_queue_linked_timeout()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/timeout.c#L693-L711)

hrtimer 是 Linux 的高精度定时器，按内核 clock 在到期时回调，不需要用户态自己轮询。io_uring 这里只保存 timeout request 和前驱 request 的关系；真正按时间排序和触发的是 hrtimer，底层 timerqueue 用 rb-tree 维护过期顺序。

前驱正常完成时，`io_disarm_next()` 会把旁边的 `IORING_OP_LINK_TIMEOUT` 从 linked chain 上摘掉，并让 timeout SQE 以 `-ECANCELED` 完成。

源码位置：
[Linux `timerqueue.c`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/lib/timerqueue.c#L3-L7)、
[Linux `timerqueue_add()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/lib/timerqueue.c#L35-L41)、
[Linux `io_disarm_next()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/timeout.c#L250-L276)

timeout 先到时，hrtimer callback 会抢走 `timeout->head`，然后通过 task_work 回到 io_uring 上下文里处理。真正的处理逻辑是：用前驱 request 的 `user_data` 构造 cancel key，调用 `io_try_cancel()`；如果 cancel 成功，timeout SQE 结果按 `-ETIME` 完成。

```c
prev = timeout->head;
timeout->head = NULL;

cd.data = prev->cqe.user_data;
ret = io_try_cancel(req->tctx, &cd, 0);
io_req_set_res(req, ret ?: -ETIME, 0);
```

源码位置：
[Linux `io_link_timeout_fn()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/timeout.c#L401-L432)、
[Linux `io_req_task_link_timeout()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/timeout.c#L366-L399)

所以 linked timeout 不是用户态 timer 的简单搬家。它把给前驱 request 标记 `REQ_F_ARM_LTIMEOUT`、启动 hrtimer、前驱正常完成时摘掉 link timeout、timeout fired 后 cancel 前驱 request 这些竞态合流放进了内核路径；`timeout.c` 维护 request 关系，时间触发交给 hrtimer。

这个特性即使在单个逻辑 op 上也有意义，比如给一次 socket read 加 deadline：

```text
SQE 1: read(socket, buffer)
SQE 2: link_timeout(1s)   // 约束这次 read
```

用户态当然可以自己做 timeout，但是复杂点不在“启动一个 timer”，而在 timeout 后提交 cancel 与原始 read completion 的竞态合流。用户态超时后提交 cancel，如果 cancel 失败，此时 read 已经完成并消费了 socket 数据；这时用户态不能简单把它当 timeout，也不能提前复用 buffer。

反过来，如果 cancel 成功，还要等对应 CQE 回来确认 read 没有按正常路径交付数据。cancel CQE、read CQE、timer callback 谁先被 event loop 处理，都会影响最终状态和资源释放路径。

linked timeout 多占一个 `IORING_OP_LINK_TIMEOUT` SQE，并在内核侧多挂一个 hrtimer/list entry；换来的是标记 `REQ_F_ARM_LTIMEOUT`、启动 hrtimer、正常完成时摘掉 link timeout、timeout 后 cancel 前驱 request 这些核心竞态由内核处理。

它不替代业务生命周期管理，buffer 和业务状态的最终回收仍然要用户态做。但对于这种固定形态的 “op + timeout”，linked timeout 能把一段本来要写进 Java 状态机的超时/取消逻辑收回到 io_uring 已经维护的 linked chain 里。

## cancel 一个 long user_data：有无 CANCEL_ALL 的差距

最后看 cancel。这个点容易被 linked shared token 误导：同一个 token 可以用来做 FIFO completion 映射，但不能自然推出“用这个 token 就能取消整批 SQE”。

liburing 准备 cancel SQE 时，会把 cancel key 放到 `sqe->addr`，把 cancel flags 放到 `sqe->cancel_flags`。真正决定“按什么匹配”和“取消一个还是全部”的是 kernel 里的 cancel 逻辑。

kernel 匹配逻辑里，如果没有指定 FD 或 OP 作为 key，就默认按 `user_data` 匹配：

```c
if (!(cd->flags & (IORING_ASYNC_CANCEL_FD | IORING_ASYNC_CANCEL_OP)))
    match_user_data = true;

if (match_user_data && req->cqe.user_data != cd->data)
    return false;
```

代码位置：
[Linux `io_cancel_req_match()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/cancel.c#L39-L68)

是否继续取消后续匹配项，取决于 `all`：

```c
bool all = cd->flags & (IORING_ASYNC_CANCEL_ALL | IORING_ASYNC_CANCEL_ANY);

do {
    ret = io_try_cancel(tctx, cd, issue_flags);
    if (ret == -ENOENT)
        break;
    if (!all)
        return ret;
    nr++;
} while (1);
```

代码位置：
[Linux `__io_async_cancel()`](https://github.com/torvalds/linux/blob/1a3746ccbb0a97bed3c06ccde6b880013b1dddc1/io_uring/cancel.c#L174-L208)

所以结果很直接：不带 `IORING_ASYNC_CANCEL_ALL` 时，kernel 按 `user_data` 找到一个匹配请求，取消一个，然后返回；带上 `IORING_ASYNC_CANCEL_ALL` 后，才会继续扫描/重试，取消所有匹配请求并返回取消数量。

多个 SQE 共享同一个 long `user_data` 时，单 cancel 不等于 cancel 这一批。

最简单的场景下，如果这 N 个同 `user_data` 的请求都还停在可取消队列里，并且期间没有 completion、没有 running、也没有新请求插入，那么提交 N 个普通 cancel 可以近似等价于一次 `IORING_ASYNC_CANCEL_ALL`：每个 cancel 命中一个请求，最终把这一批都取消掉。

但 io-wq 场景下这个等价关系不稳。pending work 还好，cancel 可以直接摘掉；如果 work 已经 running，cancel 可能只能返回 running / already 的状态，原始请求仍然按自己的路径完成，继续提交普通 cancel 也不保证会跳过这个 running 请求去取消下一个同 `user_data` 请求。

true-async 场景也类似，比如已经 poll armed 的 read。cancel 会尝试标记并唤醒对应请求，但 poll ready、原始 CQE、cancel CQE 之间仍然有竞争；用户态看到一个 cancel 成功或失败，都不能直接推导“这一批同 `user_data` 的请求已经全部取消”。

还有一个边界是期间又提交了新的同 `user_data` SQE。N 个普通 cancel 是 N 个独立请求，后面的 cancel 可能命中新提交的 SQE，而不是原来那批里的下一个；这就不再是“取消原来那一批”的语义。
