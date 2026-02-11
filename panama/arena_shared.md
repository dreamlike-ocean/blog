# 从一个关闭超时问题看Panama 内存管理API

>>  本文由排查问题时与codex聊天记录生成，并添加后期人工修正 感谢GPT-5.2-codex-high模型的支持  

## 1. 问题

原始讨论issue: https://github.com/netty/netty/issues/16174

问题背景是 Java 25 + io_uring 关闭 EventLoopGroup 变慢。提问者附上了一个火焰图，其中绝大部分时间都花费在 jdk.internal.misc.ScopedMemoryAccess.closeScope0(MemorySessionImpl, ScopedMemoryAccess$ScopedAccessError) 中，这是由 io.netty.channel.uring.IoUringIoHandler.destroy() 引起的

其中核心简化调用链如下

```
IoUringIoHandler.destroy
  -> MsgHdrMemoryArray.release
  -> MsgHdrMemory.release
  -> CleanableDirectBuffer.clean
  -> Arena.close (SharedSession.justClose)
  -> ScopedMemoryAccess.closeScope0
  -> Handshake::execute（CPP代码）
```

这里的关键是 `shared Arena close` 会触发 thread-local handshake。
而 Netty 这条链里，`IoUringIoHandler` 初始化 `MsgHdrMemoryArray(1024)`，
每个 `MsgHdrMemory` 构造时会分配 4 个 `CleanableDirectBuffer`。
在 Java 25 且 CleanerJava25 生效时，每次 `clean()` 都会 close 一个 `shared Arena`，
所以 shutdown 时会出现 `1024 * 4` 次 closeScope0。

这里导致关闭变慢的热点不是 io_uring 本身，而是 shared Arena close 的 handshake 成本。

## 2. Netty 的 Cleaner 分配方式与选择

我们先关注为什么会使用shared arean，对于netty 4.2版本来说存在四种内存分配方式：

- CleanerJava6: `ByteBuffer.allocateDirect`，通过 `sun.misc.Cleaner` 或 `DirectBuffer.cleaner` 清理。
- CleanerJava9: `Unsafe.invokeCleaner(ByteBuffer)` 清理 direct buffer。
- CleanerJava24Linker: 用 ffm api 直接调用libc的 `malloc/free`，`MemorySegment.ofAddress` 包装成 `ByteBuffer`，释放时直接 `free`。
- CleanerJava25: `Arena.ofShared` + `MemorySegment.allocate`，`clean()` 时 `Arena.close()`，也就是说一个ByteBuf对应一个shared Arena.

Netty 在 `PlatformDependent` 的选择顺序（Java 9+）：

1. CleanerJava9
2. CleanerJava24Linker
3. CleanerJava25
4. 否则 NOOP/CleanerJava6 使用gc托管的ByteBuffer

而且在jdk25上netty会强制关闭unsafe的相关路径，除非使用 `-Dio.netty.noUnsafe=false` 打开

``` java
        // See JDK 23 JEP 471 https://openjdk.org/jeps/471 and sun.misc.Unsafe.beforeMemoryAccess() on JDK 23+.
        // And JDK 24 JEP 498 https://openjdk.org/jeps/498, that enable warnings by default.
        // Due to JDK bugs, we only actually disable Unsafe by default on Java 25+, where we have memory segment APIs
        // available, and working.
        String reason = "io.netty.noUnsafe";
        String unspecified = "<unspecified>";
        String unsafeMemoryAccess = SystemPropertyUtil.get("sun.misc.unsafe.memory.access", unspecified);
        if (!explicitProperty && unspecified.equals(unsafeMemoryAccess) && javaVersion() >= 25) {
            reason = "io.netty.noUnsafe=true by default on Java 25+";
            noUnsafe = true;
        } else if (!("allow".equals(unsafeMemoryAccess) || unspecified.equals(unsafeMemoryAccess))) {
            reason = "--sun-misc-unsafe-memory-access=" + unsafeMemoryAccess;
            noUnsafe = true;
        }
```

所以在 Java 25 且 native access 未开启（--enable-native-access=ALL-UNNAMED）、Unsafe 路径不可用时，会落到 CleanerJava25。
这正是问题里触发 shared Arena close 的直接原因。

## 3. 修复方案

可以直接进行绕开

issue comment 的 workaround：

```xml
<argLine>
  --enable-native-access=ALL-UNNAMED
</argLine>
```

这会让 CleanerJava24Linker 可用，走 `malloc/free`，不再触发 shared Arena close 和 handshake。

而对于Netty侧由于我们想要不进行大改来修复这个问题 所以选择了批量化处理

`MsgHdrMemoryArray` 改成“一次分配大段 MemorySegment，然后 slice 给每个 MsgHdrMemory”。
把 4096 次 close 压到 1 次，handshake 成本直接下降。

所以这里建议大家在使用shared arena的时候一定要尽可能在arena上挂多个MemorySegment或者一口气分配大段数据以避免shared arena过多导致的握手过多问题

## 4. shared Arenas是如何触发thread-local handshake的？

关键路径在 JDK 源码：

- Java 侧：`SharedSession.justClose()` -> `ScopedMemoryAccess.closeScope(...)`
  文件：`src/java.base/share/classes/jdk/internal/foreign/SharedSession.java`
- Native 侧：`ScopedMemoryAccess_closeScope` 直接调用 `Handshake::execute`
  文件：`src/hotspot/share/prims/scopedMemoryAccess.cpp`

精简代码（Native 侧）：

```cpp
JVM_ENTRY(void, ScopedMemoryAccess_closeScope(JNIEnv *env, jobject receiver,
                                              jobject session, jobject error))
  CloseScopedMemoryHandshakeClosure cl(session, error);
  Handshake::execute(&cl);
JVM_END
```

而对于实际的握手代码——`CloseScopedMemoryHandshakeClosure` 简化代码（去掉日志和细节）：

```cpp
class CloseScopedMemoryHandshakeClosure : public HandshakeClosure {
  void do_thread(Thread* thread) {
    JavaThread* jt = JavaThread::cast(thread);
    if (!jt->has_last_Java_frame()) return;
    if (jt->has_async_exception_condition()) return;

    bool in_scoped = false;
    // 遍历栈帧看看有没有session的oop
    if (is_accessing_session(jt, session, in_scoped)) {
      // 发现正在用这个 session，注入 async exception
      jt->install_async_exception(new ScopedAsyncExceptionHandshakeClosure(session, error));
    } else if (!in_scoped) {
      // 否则 去优化 scope帧 看下是不是真的有对应的session
      frame last = get_last_frame(jt);
      if (last.is_compiled_frame() && last.can_be_deoptimized()) {
        nmethod* code = last.cb()->as_nmethod();
        // 当一个函数存在Scoped注解时就会有这个标志
        if (code->has_scoped_access()) {
          Deoptimization::deoptimize(jt, last);
        }
      }
    }
  }
};
```

`CloseScopedMemoryHandshakeClosure` 的核心行为：

- 逐个扫描 JavaThread 的 vframe
- 只关心 `@Scoped` 方法（定义在 `ScopedMemoryAccess`）
- 如果发现 thread 正在用同一个 MemorySession，注入 async exception，逼它退出 scoped access
- 如果没发现但在 compiled frame 里可能做过 scoped access，就 `deoptimize`

`@Scoped` 注解在这里：
`src/java.base/share/classes/jdk/internal/misc/X-ScopedMemoryAccess.java.template`

一个真实的 JDK 代码例子（来自 `ScopedMemoryAccess`，省略无关行）： 

当你调用这个

```java
    @ForceInline @Scoped
    private byte getByteVolatileInternal(MemorySessionImpl session, Object base, long offset) {
        try {
            if (session != null) {
                session.checkValidStateRaw();
            }
            return UNSAFE.getByteVolatile(base, offset);
        } finally {
            Reference.reachabilityFence(session);
        }
    }
```

这里的 `@Scoped` 是 VM 识别的标记，握手时只看这类方法，避免把所有 Java 方法都扫一遍。

### async exception 是什么

`async exception` 可以理解成“给另一个线程安排一个待抛出的异常”。  
线程在握手点或 safepoint poll 时抛出这个异常，然后从当前栈上展开 强制退出某一段函数。  


### thread-local handshake vs 全局 safepoint

thread-local handshake 是“逐个线程做小手术”。请求方发起后，要等所有目标线程完成，但目标线程只执行自己的 closure，不需要等其他线程：

```
requester:  发起 -> 等待全部完成
T1:         到握手点 -> 执行 closure -> 继续
T2:         到握手点 -> 执行 closure -> 继续
...
```

全局 safepoint 是“所有线程一起停下”。VMThread 等待所有线程进入 safepoint，等全局任务完成后再一起放行：

```
VMThread:   发起 -> 等待所有线程停下 -> 执行全局任务 -> 全部放行
T1/T2/...:  到 safepoint -> 停住 -> 等放行
```

区别要点：

- handshake 可以只针对部分线程，safepoint 必须是全量线程停顿
- handshake 的目标线程执行完 closure 就能继续，不需要等其他线程

所以thread-local handshake对全局影响相比于全局safepoint更小，但是并不会缩短发起握手线程的等待时间

Handshake 机制在这里：
`src/hotspot/share/runtime/handshake.cpp`

