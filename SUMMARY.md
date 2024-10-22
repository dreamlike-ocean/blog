
# Table of contents

* [概述](Readme.md)

## io_uring

* [io_uring介绍](io_uring/io_uring.md)
* [io_uring,Memory以及Procator的Socket IO](io_uring/io_uringAndMemory.md)
* [io_uring_recv是怎么实现的](io_uring/io_uring_recv_impl.md)

## reactive

* [响应式的锁](reactive_lock.md)
* [Netty4.2模型变化简介](netty-4-2.md)

## Loom

* [Spring的Loom化改造](loom/Spring的loom化改造.md)
* [为什么jvm需要有栈协程](loom/为什么jvm需要有栈协程.md)
* [虚拟线程那些事](loom/virtualThread.md)
* [VitrualThread对于synchronized适配](loom/synchronized适配.md)

## Panama

* [Panama教程-0-MethodHandle介绍](panama/panama-tutorial-0-A.md)
* [Panama教程-0-Varhandle介绍](panama/panama-tutorial-0-B.md)
* [Panama教程-1-MemorySegment介绍](panama/panama-tutorial-1-MemorySegment.md)
* [Panama教程-2-MemoryLayout介绍](panama/panama-tutorial-2-MemoryLayout.md)
* [Panama教程-3-FFI介绍](panama/panama-tutorial-3-FFI.md)
* [失去了Unsafe内存操作之后该何去何从](panama/afterUnsafe.md)
* [Panama源码浅析](panama/Panama浅析.md)

## 随笔

* [Java Memory Order和Varhandle](jmm.md)
* [基于JVM基建优化的单例模式](lazySingleton.md)

## 翻译

* [为什么现代的jvm仍然偏爱安全点](translate/【翻译】为什么现代的JVM分析器仍然偏爱安全点？.md)
* [Java虚拟机的纤程和计算续体](translate/loom_fiber_continuation.md)
* [State of Loom - Part 1](translate/state_of_loom_part1.md)
* [State of Loom - Part 2](translate/state_of_loom_part2.md)
* [loom的实现](loom/loom的实现.md)

## CPP

* [C++20协程入门](cpp_coroutine/first.md)
* [async scope和通用回调转协程](cpp_coroutine/async_scope.md)