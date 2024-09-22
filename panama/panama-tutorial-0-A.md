# Panama教程-0-MethodHandle介绍

## 前言

在介绍Panama用法之前，需要引入一个关于MethodHandle的介绍，Panama中会大量用到这个类

简单来说Methodhandle看起来像是一个reflection api的类似物，但是实际上它比反射更加底层同时JIT对其的优化更加良好，其所在的`java.lang.invoke`是一组强大的代码生成和方法动态调用的工具

##  `MethodHandle` （方法句柄）简介

### 什么是方法句柄

从jdk的注释来看

```txt
A method handle is a typed, directly executable reference to an  underlying method, constructor, field, or similar low-level operation, with  optional transformations of arguments or return values
```

- **类型化**：方法句柄是有类型的，意味着它知道自己操作的参数和返回值的类型。
- **直接可执行**：方法句柄可以直接执行，不需要额外的解释或编译步骤。
- **低级别操作**：方法句柄引用的是底层的方法、构造函数、字段或类似的低级操作。
- **可选转换**：方法句柄支持对参数或返回值进行自定义的转换。

在使用过程中还有一个特点是immutable——不可变，对于方法句柄的转换会产生一个新的方法句柄，在其上进行调用也也能保证无法观测到前后状态不一致

如果你对C之类的语言比较了解，其实一个简单的方法句柄其实就是一个函数指针

### 如何获取并使用方法句柄

最简单的方案就是使用 `MethodHandles::lookup` 工厂方法获取lookup。

然后，我们可以使用 `Lookup` 中的 `findXXX` 方法之一来查找现有的 Java 方法并调用

```java
    public static void main(String[] args) throws Throwable {
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        MethodHandle CONCAT_MH = lookup.findVirtual(String.class, "concat", MethodType.methodType(String.class, String.class));
        assertEquals("hello world", (String) CONCAT_MH.invokeExact("hello", " world"));
    }

```

`Lookup` 中的其他 `findXXX` 方法可用于查找其他类型的方法。`findVirtual` 方法对应于 `invokevirtual` 字节码，可用于查找实例（即非静态）方法，也称为*虚方法*。`findConstructor` 可用于查找构造函数。还有 `find（Static）Getter` 和 `find（Static）Setter` 用于读取和写入字段。请注意，这些*方法不*查找 `getX` 或 `setX` 方法，而是查找直接获取或设置字段的名义方法。这就像一个动态生成的获取或设置字段的方法。这些字节码对应于 `putfield`、`getfield`、`putstatic` 和 `getstatic` 字节码。

> 对于findVirtual其会在获得的句柄最前面添加一个receiver对象 用于多态实现

这就是他的**低级别操作**的特性

### 方法句柄使用须知

#### 权限控制

这里我们就要跟reflection api对比了

对于`Method::invoke`很明显每次调用时会检查调用方权限，这一段开销是无法省略的，排除对于权限的检查，仔细看下CallerSensitive这个注解，这个注解是jvm内部用来获取调用者堆栈的，为了防止[双重反射构建特权调用的情况](https://stackoverflow.com/questions/22626808/what-does-the-sun-reflect-callersensitive-annotation-mean)，其会遍历堆栈找到真实调用者，这一点也是比较消耗性能的

```java
    @CallerSensitive
    @ForceInline // to ensure Reflection.getCallerClass optimization
    @IntrinsicCandidate
    public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, InvocationTargetException
    {
        boolean callerSensitive = isCallerSensitive();
        Class<?> caller = null;
        if (!override || callerSensitive) {
            caller = Reflection.getCallerClass();
        }

        // Reflection::getCallerClass filters all subclasses of
        // jdk.internal.reflect.MethodAccessorImpl and Method::invoke(Object, Object[])
        // Should not call Method::invoke(Object, Object[], Class) here
        if (!override) {
            checkAccess(caller, clazz,
                    Modifier.isStatic(modifiers) ? null : obj.getClass(),
                    modifiers);
        }
        //。。。。省略其余代码
    }
```

方法句柄在调用时不执行任何访问检查（因此不需要区分调用方）。相反，访问检查是在查找方法时执行的。你会注意到 `MethodHandles::lookup` 的方法被标注了CallerSensitive注解。

```java
    @CallerSensitive
    @ForceInline // to ensure Reflection.getCallerClass optimization
    public static Lookup lookup() {
        final Class<?> c = Reflection.getCallerClass();
        if (c == null) {
            throw new IllegalCallerException("no caller frame");
        }
        return new Lookup(c);
    }
```

调用 `MethodHandles::lookup` 时，caller 类被捕获放进 **lookup** 类实例里面。然后，在执行方法句柄查找时，以caller是否具有对应方法的权限进行判断。这允许在创建方法句柄时执行单次访问检查，然后通过在调用方法句柄时避免访问检查来获得更好的性能。

如何理解 `caller是否具有对应方法的权限`呢？简单来说就是 在caller中是否可以直接调用对应的方法，你可以想象一下，你正在caller对应的类中写代码，能通过写代码直接调用的方法都算作你持有对应的访问权限。

举个例子：

```java
    public static void main(String[] args) throws Throwable {
        // 可以调用 因为这个Lookup 对应的PrivateClass类可以访问PrivateClass::privateMethod
        PrivateClass.LOOKUP.findStatic(PrivateClass.class, "privateMethod", MethodType.methodType(void.class))
                .invokeExact();

        // 不可以调用 因为这个Lookup 对应的Main类无法访问PrivateClass::privateMethod
        MethodHandles.lookup()
                .findStatic(PrivateClass.class, "privateMethod", MethodType.methodType(void.class))
                .invokeExact();
    }

```

#### 签名多态

这个名字很奇怪，签名多态（[Signature polymorphism](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/MethodHandle.html#sigpoly)）因为我实在不知道该怎么翻译了

我们先来看一下reflection api和methodhandle api的对比

```java
public Object invoke(Object obj, Object... args);
public final native @PolymorphicSignature Object invokeExact(Object... args)
```

嗯？好像看起来差不多？

这就要聊到PolymorphicSignature这个注解了，它的含义是我们生成出来的调用字节码并不是根据方法签名生产的而是根据调用点传参类型声明的，似乎有点不好理解？

对于一个普通的方法`void assertEquals(Object expected, Object actual)`其生成的字节码为`INVOKESTATIC io/github/dreamlike/Main.assertEquals(Ljava/lang/Object;Ljava/lang/Object;)V` 是完全根据方法的签名生成的

而对于签名多态则是根据调用点如何传入参数生成的对应字节码

```java
    public static void polymorphicSignature(MethodHandle methodHandle) throws Throwable {
//INVOKEVIRTUAL java/lang/invoke/MethodHandle.invokeExact (Ljava/lang/String;I)Ljava/lang/Long;
        Long res = (Long) methodHandle.invokeExact((String) null, 1);
//INVOKEVIRTUAL java/lang/invoke/MethodHandle.invokeExact (Ljava/lang/Integer;Ljava/lang/Long;)V
        methodHandle.invokeExact(Integer.valueOf(12), (Long)20L);
    }
```

对于同一个方法的调用生成出来的调用字节码完全不一样 这就是签名多态——由调用点决定调用参数列表和返回值

那么他有什么好处呢？还是要回去看reflection api

调用 `Method::invoke` 方法需要创建一个新的 `Object[]`，将所有参数值存储到其中，如果涉及原始类型，甚至还需要装箱。而签名多态可以让我们原封不动地转发参数到对应的方法上，并且返回值也不需要做改动。

然后我们需要再谈谈methodhandle的两个签名多态函数`invoke`和`invokeExact`

invokeExact会原封不动地将调用点的参数多态跟实际调用点的方法签名进行匹配，若不匹配则直接抛出异常，这里就是上面提到的**类型化**的特征，每一个方法句柄都可以通过`methodType`方法获取当对应的类型

invoke虽然也是签名多态的但是并不强求签名一致，他会根据每次实际调用点的类型，生成一些字节码将传入的参数做一层转换后再实现invokeExact

举个简单的例子

```java
    static void foo(Object o) {}

    public static void main(String[] ___) throws Throwable {
        MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
                MethodType.methodType(void.class, Object.class));
        fooMh.invoke("Hello, world!"); // 可以调用 会自动插入转换
        fooMh.invokeExact("Hello, world!"); //不可以调用 提示handle's method type (Object)void but found (String)void
    }

```

那么为什么要使用invokeExact而非invoke呢？

`invoke` 的实现将调用 `MethodHandle::asType` 方法,记得我们之前提过的**不可变特性**么？asType实际上会生成生成一个合成类 + 方法，用于实现所有参数和返回类型转换，这部分的生产和转换的开销对于单纯调用一个函数来讲还是很大的，但是并不一定每次都会重新生成，当我们仔细看下源码时，会发现有一个缓存机制。也就是说仅当使用不同的调用点类型多次调用同一方法句柄实例时，才会生成。

```java
    public final MethodHandle asType(MethodType newType) {
        // Fast path alternative to a heavyweight {@code asType} call.
        // Return 'this' if the conversion will be a no-op.
        if (newType == type) {
            return this;
        }
        // Return 'this.asTypeCache' if the conversion is already memoized.
        MethodHandle at = asTypeCached(newType);
        if (at != null) {
            return at;
        }
        return setAsTypeCache(asTypeUncached(newType));
    }

```

如果我们可以确保调用类型时，就可以使用invokeExact规避掉这个桥接生成和其带来的自动类型转换。

对于实践中我们更推荐将其包装一下进行调用

```java
static final MethodHandle FOO_MH = ...

public static void main(String[] ___) throws Throwable {
    fooWrapper("Hello, world!");
}

public static void fooWrapper(Object o) {
    FOO_MH.invokeExact(o);
}
```

#### 性能问题

当这个methodhandle并非是const的时候（一般是static + final）并不能被JIT正确折叠以达到内联到真实调用点的情况，所以还是推荐大家尽可以将其声明为const且使用invokeExact

如果实在是做不到const,比如说你想惰性链接，对应的methodhandle想要惰性初始化又想要性能那么其实有几个方案适合你

##### LambdaMetafactory方案

这就是Java 的lambda实现 我们借用过来在需要调用的时候直接把methodhandle转换为函数接即可

```java
(Add) LambdaMetafactory.metafactory(
                    MethodHandles.lookup(),
                    "apply",
                    MethodType.methodType(Add.class),
                    MethodType.methodType(int.class, int.class, int.class),
                    ADD_MH,
                    ADD_MH.type()
            ).getTarget().invokeExact();
```

##### invokedynamic + hidden class方案

将惰性获取的Supplier接口通过hidden class的classdata传递给对应的bootstrap方法，在第一次调用对应方法的时候惰性去解析调用点

```java
    public static Add generate(Supplier<MethodHandle> supplier) throws IllegalAccessException, InstantiationException {
        byte[] classByteCode = ClassFile.of()
                .build(ClassDesc.of(LambdaBenchmarkCase.class.getName() + "AddImpl"), cb -> {
                    //省略不重要代码
                    cb.withMethodBody("apply",
                            MethodTypeDesc.of(CD_int, CD_int, CD_int),
                            AccessFlags.ofMethod(AccessFlag.PUBLIC, AccessFlag.SYNTHETIC).flagsMask(),
                            it -> {
                                it.iload(1);
                                it.iload(2);
                                it.invokeDynamicInstruction(
                                        DynamicCallSiteDesc.of(
                                                ConstantDescs.ofCallsiteBootstrap(LambdaBenchmarkCase.class.describeConstable().get(), "indyLambdaFactory", ConstantDescs.CD_CallSite),
                                                "apply",
                                                MethodTypeDesc.of(CD_int, CD_int, CD_int)
                                        )
                                );
                                it.returnInstruction(TypeKind.IntType);
                            });
                });

        MethodHandles.Lookup lookup = MethodHandles.lookup()
                .defineHiddenClassWithClassData(classByteCode, supplier, true);

        return (Add) lookup.lookupClass().newInstance();
    }

    public static CallSite indyLambdaFactory(MethodHandles.Lookup lookup, String name, MethodType type) throws NoSuchFieldException, IllegalAccessException {
        MethodHandle methodHandle = ((Supplier<MethodHandle>) MethodHandles.classData(lookup, ConstantDescs.DEFAULT_NAME, Supplier.class)).get();
        return new ConstantCallSite(methodHandle);
    }
```

##### MutableCallSite方案

参考[该回答](https://stackoverflow.com/questions/49292210/is-it-possible-to-make-java-lang-invoke-methodhandle-as-fast-as-direct-invokatio)以及 `MutableCallSite::syncAll`

```java
class Holder {
    private static final MutableCallSite callSite = new MutableCallSite(
            MethodType.methodType(int.class, int.class, int.class)
    );
    private static final MethodHandle callSiteInvoke = callSite.dynamicInvoker();

    
    public int call(int a, int b) {
        return (int)callSiteInvoke.invokeExact(a,b);
    }

    public static void lazySet(MethodHandle target) {
        callSite.setTarget(target);
        VarHandle.releaseFence();
        Objects.requireNonNull(callSite); //构成callSite引用的读写顺序同步。。
    }
}
```

对应代码和测试可以参考这个

<script src="https://gist.github.com/dreamlike-ocean/088b55f552353f71e71a5e34f6dfdef3.js"></script>

## 参考资料

[MethodHandle primer](https://jornvernee.github.io/methodhandles/2024/01/19/methodhandle-primer.html)

[R大的Methodhandle介绍](https://greenteajug.cn/images/%E8%8E%AB%E6%9E%A2_201207-LF-Tutorial.pdf)

[oracle的详细的mh介绍文档](https://cr.openjdk.org/~vlivanov/talks/2015-Indy_Deep_Dive.pdf)

[更有趣的jit剪枝](https://stackoverflow.com/questions/53271900/jit-micro-optimization-if-statement-elimination)