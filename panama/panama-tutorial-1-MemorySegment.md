# Panama教程-1-MemorySegment介绍

## 前言

对于Panama来说其分为两部分，一部分是FFM,即一些内存操作的API,一部分是FFI,即如何调用符合当前平台的ABI的动态库代码

本文是对于FFM部分的介绍

## MemorySegment基础介绍

粗看MemorySegemnt与NIO包下的Bytebuffer差不多，都是支持将byte[]和堆外指针封装为一个对象，实现在统一视图下的操作

### 使用

其并不提供任何分配内存的api,而是专注于包装已有的内存指针

其堆外内存的分配来于SegmentAllocator接口的实现类，至于为什么需要Arena类我们下面再展开

```java
// 包装一个Java原始类型数组
MemorySegment.ofArray(...); 

//分配一段堆外内存
Arena arena = Arena.ofConfined();
MemorySegment allocate = arena.allocate(8);

//包装一个堆外指针
MemorySegment.ofAddress(...);

//包装一个nio bytebuffer
MemorySegment.ofBuffer(ByteBuffer.allocate(8))
```

对于内存读写操作也很简单

就像是指针操作一样直接从某个偏移量获取值即可，这里是跟ByteBuffer一个很大的不同，ByteBuffer会维护一组“内部状态”，你需要在各种get/put/flip方法之间辗转腾挪比较麻烦，而MemorySegment的api则很干净，只支持简单的基于偏移量的取值

```java
memorySegment.get(ValueLayout.JAVA_INT, /*offset*/0);
memorySegment.set(ValueLayout.JAVA_INT, /*offset*/0, /*value*/1);
memorySegment.asSlice(/*offset*/0, /*newSize*/12);
```

### 与ByteBuffer区别

#### 堆外内存生命周期

> 当我们谈论这里的生命周期的时候一般是谈论他的malloc和free
>
> 而且若无提及，均指的是public api

ByteBuffer的堆外内存的生命周期，分配是交由用户手动分配的，但是释放却是交由GC释放，我们并不能手动干涉将其立刻释放

而MemorySegment则是将其生命周期与Arena相绑定，Arena这个词也是个懂得都懂的词，不懂的还是看不懂的词，他实际上可以理解为一个Scope,将MemorySegment的释放回调挂载到这个Scope上，当这个Scope关闭时，释放掉由它分配的全部MemorySegment

```java
        try (Arena scope = Arena.ofConfined()) {
            MemorySegment memorySegment = scope.allocate(1024);
            //do something
        } // 在这里面被释放
```

#### 字节序

对于ByteBuffer而言其分配出来的堆外内存，在使用ByteBuffer API进行访问的时候均为大段序

而MemorySegment为native order。

以我当前使用的intel x86_64bit的Linux主机为例，当前平台的native order为小端序

当我指定对应byte order访问的时候 直接getLong的值才与MemorySegment一致

```java
        ByteBuffer byteBuffer = ByteBuffer.allocateDirect(8);
        assertEquals(byteBuffer.order(), ByteOrder.BIG_ENDIAN);

        MemorySegment memorySegment = MemorySegment.ofBuffer(byteBuffer);
        memorySegment.set(ValueLayout.JAVA_LONG, 0, 1);
        assertEquals((long) 1 << 56, byteBuffer.getLong(0));
        assertEquals(memorySegment.get(ValueLayout.JAVA_LONG, 0), byteBuffer.order(ByteOrder.LITTLE_ENDIAN).getLong(0));
```

## Arena和MemorySegment

> 若无特意提及则默认均为堆外内存

这里我们简单介绍下我们刚才草草省略过的Arena，也就是MemorySegment的生命周期管理

### 统一概念

Arena这里本意是场馆的意思，实际上就是暗示它管理了全部由其分配的内存段的生命周期。

注意虽然Golang之类的语言中也有Arena但是Java这里的Arena并不保证每次分配出来的内存是连续的，对于某些语言Arena api只是将一大块内存拿出来切块分配，最后统一回收，但是Java仅仅是强调统一回收这一概念。

除了我们在上面展示的通过close回收分配的堆外内存之外，还会提供在Arena close后限制对应内存段的读写操作。如下代码，当我们进行操作一个关联了scope的内存端时，其scope被关闭后，会直接抛出异常，避免出现UAF的高危行为

```java
    public static void main(String[] args) throws Throwable {
        Arena localScope = Arena.ofConfined();
        MemorySegment segment = localScope.allocate(1024);
        localScope.close();
//Exception in thread "main" java.lang.IllegalStateException: Already closed
        segment.get(ValueLayout.JAVA_INT, 0);

    }
```

Arena根据实现不同，还能提供Thread Owner的控制，那么我们接下来介绍下几个常用的Arena

### Confined Arena

这是一个限定比较多的也是比较常用的一个Arena,其返回的Arena是一个只允许单线程使用的且需要手动关闭的Arena

```java
    public static void main(String[] args) throws Throwable {
        Arena localScope = Arena.ofConfined();
        MemorySegment segment = localScope.allocate(1024);

        Thread.startVirtualThread(() -> {
            try {
                localScope.allocate(1024);
            } catch (Throwable t) {
                System.out.println("其他线程无法分配");
            }
            try {
                segment.get(ValueLayout.JAVA_INT, 0);
            } catch (Throwable t) {
                System.out.println("其他线程无法访问");
            }
        }).join();

        localScope.close();
        try {
            segment.get(ValueLayout.JAVA_INT, 0);
        } catch (Throwable t) {
            System.out.println("关闭后无法使用");
        }

    }
```

### Shared Arena

这是一个多线程版本的Confined Arena，允许在其余线程中进行访问，但是需要手动关闭，在关闭过程中若仍有线程在使用其分配出来的内存段则会关闭失败，建议与结构化并发API一同使用

```java
    public static void main(String[] args) throws Throwable {
        MemorySegment uaf = null;
        try (StructuredTaskScope<Void> currencyScope = new StructuredTaskScope<Void>();
             Arena arena = Arena.ofShared()
        ) {
            StructuredTaskScope.Subtask<Void> task1 = currencyScope.fork(() -> {
                MemorySegment _ = arena.allocate(1024);
                return null;
            });

            StructuredTaskScope.Subtask<Void> task2 = currencyScope.fork(() -> {
                MemorySegment _ = arena.allocate(1024);
                return null;
            });
            currencyScope.join();
            assertEquals(task1.state(), StructuredTaskScope.Subtask.State.SUCCESS);
            assertEquals(task2.state(), StructuredTaskScope.Subtask.State.SUCCESS);

            uaf = arena.allocate(1024);
        }
        //抛出异常
        uaf.get(ValueLayout.JAVA_BYTE, 0);
    }
```

### Auto Arena

这是一个**不允许手动关闭**的Shared Arena，允许在其余线程中进行访问，其分配的内存端均被自动管理——其实就是GC机制+Cleaner管理Auto Arena的生命周期，这里并不是让GC管理每一个内存段而是通过发现Auto Arena不可达之后，触发一个cleaner回调将其分配的全部内存段回收

对于其分配出来的MemorySegment，均会持有对应Auto Arena的引用，即 MemorySegment <=> Arena，这样只有Arena不可达，且其分配出来的每一个MemorySement均不可达 才允许其管理的全部MemorySegment被释放

某种程度上这种管理方案与旧有的ByteBuffer 管理方式一致

```
    public static void main(String[] args) throws Throwable {
        Arena auto = Arena.ofAuto();
        MemorySegment memorySegment = auto.allocate(ValueLayout.JAVA_INT);
    }
```

### Global Arena

这个就更简单了，它不允许关闭，只管分配也不进行任何回收。类似于Rust的`static生命周期，其分配出来的内存允许跨线程，允许任意使用，随着进程销毁一起回收

大部分情况下可能没什么用，但是`MemorySegment.ofAddress(0)`中，就默认是Global Arena，对于大部分从native返回的指针均为这个作用域，所以可以通过这个API来 “擦除” 作用域

```java
    public static void main(String[] args) throws Throwable {
        Arena global = Arena.global();
        System.out.println(global.allocate(1024).scope());
        System.out.println(MemorySegment.ofAddress(0).scope());
    }
```

### 自定义Arena

你可以使用` java.lang.foreign.SegmentAllocator`实现一个将一大块内存拿出来切块分配，最后统一回收的Arena,batch化内存分配操作

```java
    class SlicingArena implements Arena {
        final Arena arena = Arena.ofConfined();
        final SegmentAllocator slicingAllocator;

        SlicingArena(long size) {
            slicingAllocator = SegmentAllocator.slicingAllocator(arena.allocate(size));
        }

        public MemorySegment allocate(long byteSize, long byteAlignment) {
            return slicingAllocator.allocate(byteSize, byteAlignment);
        }

        public MemorySegment.Scope scope() {
            return arena.scope();
        }

        public void close() {
            arena.close();
        }

    }
```

### Arena关闭时的内存安全

#### 引用计数法

刚才提到了在Arena关闭后无法通过MemorySegment相关的API进行访问，但是在我们跟操作系统API交互的时候，OS无法感知我们这个检测机制，那么是否意味着我们在使用Java提供的File IO等FFI过程中内存存在安全问题呢？

实际上并不会，因为对于那些允许close的Arena他们还有个引用计数法来在关闭时检测是否活跃

举个例子对于FileChannel来讲 Java通过IOUtil类统一封装了访问方式，统一收口处增加检测代码，用伪代码大概是这样的

```java
segemnt.arena.count ++;
file.read(segment);
segement.arena.count--;
```

在关闭时则直接检测count的值，若不为0说明仍在活跃，不应该关闭

我们可以通过Socket API构造一个永不返回的系统调用，来看看具体的close效果

```java
    public static void main(String[] args) throws Throwable {
        Thread.startVirtualThread(() -> {
            try(ServerSocket socket = new ServerSocket(8300);) {
                Socket s = socket.accept();
                System.out.println("Accept: " + s);
                //只接受不读取
                LockSupport.park();
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        });
        //wait listen
        TimeUnit.SECONDS.sleep(1);
        Arena arena = Arena.ofShared();
        try (arena) {
            MemorySegment segment = arena.allocate(1024);
            ByteBuffer byteBuffer = segment.asByteBuffer();
            new Thread(() -> {
                try {
                    SocketChannel channel = SocketChannel.open();
                    boolean connect = channel.connect(new InetSocketAddress("127.0.0.1", 8300));
                    assertEquals(connect, true);
                    //wait infinite
                    channel.read(byteBuffer);
                    System.out.println("end read");
                    assertEquals(byteBuffer.get(0), 1);
                    channel.close();
                } catch (Throwable e) {
                    e.printStackTrace();
                }
            }).start();
            TimeUnit.SECONDS.sleep(1);
        }
    }
```

最后clonse会抛出这样一个异常

```java
Exception in thread "main" java.lang.IllegalStateException: Session is acquired by 1 clients
```

最后一提new Thread这里不要替换为使用虚拟线程，由于虚拟线程发起read操作是在poller唤醒线程之后而不是在调用read的时候，所以换成虚拟线程你只能在虚拟线程那边拿到一个`Already closed`的异常

#### 线程本地握手

shared与confined的最大不同在与前者允许多线程访问，对于一个普通的访存操作都去增加一个引用计数显然是不现实的，所以shared在检测引用计数之后若发现其引用计数为0也不代表没有线程在访问这块内存了。

我们可以之直接看下ShareSession（这个就是具体的实现）是如何做的

1,第一步cas一下，设置为close这样在close之后的访问操作就无法进行，限制新增

2,发起closeScope处理现存的内存操作

```java
    void justClose() {
        int prevState = (int) STATE.compareAndExchange(this, OPEN, CLOSED);
        if (prevState < 0) {
            throw alreadyClosed();
        } else if (prevState != OPEN) {
            throw alreadyAcquired(prevState);
        }
        SCOPED_MEMORY_ACCESS.closeScope(this, ALREADY_CLOSED);
    }
```

那么如何找出来哪些线程还在访问这段内存呢？Java对其的实现是依次查看JavaThread的栈是否正在调用对应操作，那么这里就有两个问题

1,如何高效且安全枚举JavaThread

2,如何判断某个线程正在访问这段内存

##### 如何高效且安全枚举JavaThread

一个JavaThread的栈帧最稳定的时候是它处于安全点的时候，这个时候我们可以安全地枚举每一帧，处理每一帧里面的oop,但是为了枚举某一个线程就使用**全局安全点**这个操作太重了，所以它其实利用的机制叫做**[线程本地握手](https://openjdk.org/jeps/312)**，依次去停顿对应的线程，相当于某个线程poll的其实是某个ThreadLocal的安全点，然后在终端处理函数中执行对应回调，这样在回调里面可以轻松枚举当前线程栈上的东西

对应代码可以参考[CloseScopedMemoryClosure](https://github.com/openjdk/jdk/blob/7cc7c080b5dbab61914512bf63227944697c0cbe/src/hotspot/share/prims/scopedMemoryAccess.cpp#L131)这个类中的do_thread方法

##### 判断正在访问这段内存

以getInt为例子它其实就是一个Unsafe的封装，只不过这个方法被`Scoped`标注，这个Scope会被JVM特殊处理用于给某一栈帧打上一个特殊标识

```java
    @ForceInline @Scoped
    private int getIntInternal(MemorySessionImpl session, Object base, long offset) {
        try {
            if (session != null) {
                session.checkValidStateRaw();
            }
            return UNSAFE.getInt(base, offset);
        } finally {
            Reference.reachabilityFence(session);
        }
    }
```

我们可以直接来看看核心的代码（经过部分修改 更容易阅读）

```cpp
static bool is_accessing_session(JavaThread* j, /*sharedsession对象*/oop session) {
  ResourceMark rm;
  const int max_critical_stack_depth = 10;
  int depth = 0;
    //遍历每一帧
  for (vframeStream stream(jt); !stream.at_end(); stream.next()) {
    Method* m = stream.method();
    bool is_scoped = m->is_scoped(); /这里就是我们刚才提到的Scoped注解标注
    if (is_scoped) {
        //获取这一帧上全部的oops 如果有对应的session则认为找到了正在使用的
        //此时不应该close
     StackValueCollection* locals = stream.asJavaVFrame()->locals();
     for (int i = 0; i < locals->size(); i++) {
      StackValue* var = locals->at(i);
      if (var->type() == T_OBJECT) {
        if (var->get_obj() == session) {
          return true;
        }
      }
      }
     return false;
    }
    depth++;
  }
  return false;
}
```

这样就做到了安全关闭shared arena，关闭增量，探测存量