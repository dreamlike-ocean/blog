# Panama教程-2-MemoryLayout介绍

> 若无特别提及，这里的数据类型长度均为x86 64 linux上的长度
>
> 示例代码可以从这里找到
>
> https://github.com/dreamlike-ocean/Panama-tutorial

## 前言

之前我们介绍了MemorySegement的相关内容，那么在迈向FFI的介绍之前，我们先来看一个东西MemoryLayout,它用来描述与FFI交互的时候应该传入什么类型的参数，你可以简单理解为这就是一个c语言中的结构体

## 基础概念

我们先来看一下MemoryLayout的功能

蓝色框只是一些对与当前的MemoryLayout的元数据描述，以及一些可以给它打tag的api

红色的的部分则是MemoryLayout提供的各种方便操作内存的API——获取偏移量，获取操作偏移量的varhandle

![ecde3c69-6498-4205-bc34-342d2207112b](assets/ecde3c69-6498-4205-bc34-342d2207112b.jpeg)

## 基础类型

对于八大基础类型和指针类型均有对应的的MemoryLayout实现，你可以通过` ValueLayout.JAVA_INT`获取到一个基础类型的MemoryLayout对象

```java
public sealed interface ValueLayout extends MemoryLayout
        permits ValueLayout.OfBoolean, ValueLayout.OfByte, ValueLayout.OfChar,
        ValueLayout.OfShort, ValueLayout.OfInt,  ValueLayout.OfFloat,
        ValueLayout.OfLong, ValueLayout.OfDouble, AddressLayout
```

这里有个小坑，对于c而言char的长度为一个字节，但是Java的char长度为2个字节，c的char对应的是java的byte

它的用法一般是这样的，通过某个偏移量从某个MemorySegment获取一个Java_Int的值，当你获取到一个varhandle时就可以通过varhandle api+偏移量形式获取对应位置的值

```java
    public static int getInt(MemorySegment memorySegment) {
        VarHandle varHandle = ValueLayout.JAVA_INT.varHandle();

        return (int) varHandle.get(memorySegment,/*offset*/ 0);
    }

```

大部分情况下我们都是从0偏移量的地方进行获取，所以我们可以这样转换下Varhandle，正如我们之前文章讲到的，Varhandle都是不可变的所以这里转换并不会影响到原始的Varhandle,而是返回一个新的Varhandle

```java
  private static final VarHandle GET_INT_OFFSET_0 = MethodHandles.insertCoordinates(ValueLayout.JAVA_INT.varHandle(), 1, 0);

```

以上是针对于对齐的内存进行操作，所以若传入的内存未对齐，那么就会抛出一个异常

```java
IllegalArgumentException illegalArgumentException = Assertions.assertThrows(IllegalArgumentException.class, () -> {
    var memorySegment1 = scope.allocate(ValueLayout.JAVA_INT.byteSize() + 1);
    alueLayout.JAVA_INT.varHandle().set(memorySegment1, 1, 2001);
});
 //java.lang.IllegalArgumentException: Target offset 1 is incompatible with alignment constraint 4 (of i4) for segment MemorySegment{ address: 0x707c9442e020, byteSize: 5 }

```

如果你有确切把握你所在的平台支持未对齐读写，那么就可以使用对应的`ValueLayout.*_UNALIGNED`进行读写操作

```java
var memorySegment1 = scope.allocate(ValueLayout.JAVA_INT.byteSize() + 1);
Assertions.assertEquals(2001, ValueLayout.JAVA_INT_UNALIGNED.varHandle().get(memorySegment1, 1));
```

注意未对齐读写可能存在不可预知的问题，甚至会导致jvm crash,请务必三思后行

## 数组类型

对于一个数组类型，它是n个具有相同MemoryLayout连续排列的一个连续内存布局，n为声明描述时制定的一个变量

参考以下代码 即可以获取到一个varhandle用来遍历这个数组

```java
public static int sum(MemorySegment intArray) {
    int count = (int) (intArray.byteSize() / ValueLayout.JAVA_INT.byteSize());
    SequenceLayout sequenceLayout = MemoryLayout.sequenceLayout(/*count*/count, ValueLayout.JAVA_INT);
    //真实使用的时候务必将VarHandle const化
    VarHandle varHandle = sequenceLayout.varHandle(MemoryLayout.PathElement.sequenceElement());
    int sum = 0;
    for (int i = 0; i < count - 1; i++) {
        sum += (int) varHandle.get(intArray, 0, i);
    }
    //专门用来获取最后一个元素 同时给 VarHandle 插入基准偏移量 0
    VarHandle indexVarhandle = MethodHandles.insertCoordinates(sequenceLayout.varHandle(MemoryLayout.PathElement.sequenceElement(count - 1)), 1, 0);
    sum += (int) indexVarhandle.get(intArray);
    return sum;
}
```

## 填充类型

`MemoryLayout.paddingLayout`这是一个没有实际字段意义的的类型，只是用来填充对齐空间的，这里不展开，在结构体类型中经常使用

## 结构体/联合类型

### 声明结构体

与native打交道比较重要的类型就是结构体类型,下面这段Rust代码声明了一个c布局的结构体，第一个字段是32位的int类型，第二个是64位的int类型

```rust
#[repr(C)]
pub struct Person {
    pub a: i32,
    pub n: i64,
}
```

那么是否我们可以直接这样声明一个Java侧的结构体呢？

```java
StructLayout structLayout = MemoryLayout.structLayout(
        ValueLayout.JAVA_INT.withName("a"),
        ValueLayout.JAVA_LONG.withName("n")
);
```

如果你真的这样写了就会发现他会抛出一个异常`java.lang.IllegalArgumentException: Invalid alignment constraint for member layout: j8(n)`

通过这个异常我们可以看出来Java这个structLayout的实现非常简单粗暴，只是单纯依次放置对应的MemoryLayout罢了

为了让其能够通过运行时检查我们需要这样写，在a和n之间插入一个四个字节长的填充类型

```java
public static final MemoryLayout PERSON_LAYOUT = MemoryLayout.structLayout(
        ValueLayout.JAVA_INT.withName("a"),
        MemoryLayout.paddingLayout(4),
        ValueLayout.JAVA_LONG.withName("n")
);
```

### 声明联合（Union）

学过c的读者应该能意识到这个联合类型是并不需要struct那样的对齐策略

```java
public static final MemoryLayout UNION_SAMPLE_STRUCT_LAYOUT = MemoryLayout.unionLayout(
        ValueLayout.JAVA_INT.withName("a"),
        ValueLayout.JAVA_LONG.withName("n")
);
```

此时这个union的长度为一个Java_Long的长度

### 操作字段

那么我们该如何操作对应的字段呢？

```java
 public static long getN(MemorySegment memorySegment, boolean useName) {
     VarHandle varHandle = useName
             //由于我们使用了ValueLayout.JAVA_LONG.withName("n") 所以可以通过名字获取varhandle
             ? PERSON_LAYOUT.varHandle(MemoryLayout.PathElement.groupElement("n"))
             // 由于名字为n的字段是第三个布局元素（包含填充类型的布局） 所以这里是2
             : PERSON_LAYOUT.varHandle(MemoryLayout.PathElement.groupElement(2));
     //老样子习惯性插入基准偏移量
     varHandle = MethodHandles.insertCoordinates(varHandle, 1, 0);
     return (long) varHandle.get(memorySegment);
 }
```

通过使用一组PathElement来指定对应的搜索路径以获取Varhandle,同理你也可以使用`PERSON_LAYOUT.byteOffset`获取某一字段的偏移量

### 稍微复杂一点的例子

下面我给一个包含嵌套结构体，数组的例子，为了方便展示我故意将全部的类型都设置为long类型，这样可以不需要计算对齐策略

其依次为一个long类型字段，一个长度为3的long数组，一个包含两个long类型的嵌套结构体，一个指向某个long数组的指针，一个指向long_and_long的指针

```rust
#[repr(C)]
pub struct long_and_long {
    pub a: i64,
    pub b: i64,
}

#[repr(C)]
pub struct Complex {
    pub a: i64,
    pub long_array: [i64; 3],
    pub sub_struct: long_and_long,
    pub long_array_ptr: *mut i64,
    pub long_and_long_ptr: *mut long_and_long,
}
```

### 布局声明

那么他该如何声明布局呢？

```java
private static final MemoryLayout LONG_AND_LONG_LAYOUT = MemoryLayout.structLayout(
        ValueLayout.JAVA_LONG.withName("a"),
        ValueLayout.JAVA_LONG.withName("b")
);
public static final MemoryLayout COMPLEX_LAYOUT = MemoryLayout.structLayout(
        ValueLayout.JAVA_LONG.withName("a"),
        MemoryLayout.sequenceLayout(3, Integer.JAVA_LONG).withName("long_array"),
        LONG_AND_LONG_LAYOUT.withName("sub_struct"),
        ValueLayout.ADDRESS.withTargetLayout(MemoryLayout.sequenceLayout(Long.MAX_VALUE, ValueLayout.JAVA_LONG)).withName("long_array_ptr"),
        ValueLayout.ADDRESS.withTargetLayout(LONG_AND_LONG_LAYOUT).withName("long_and_long_ptr")
);
```

### 获取long_array的第二个元素

还是通过一组PathElement灵活组织下就好了

```java
 public static long getLongArrayIndex1(MemorySegment memorySegment) {
     VarHandle varHandle = COMPLEX_LAYOUT.varHandle(
             MemoryLayout.PathElement.groupElement("long_array"),
             MemoryLayout.PathElement.sequenceElement(/*index*/ 1)
     );
     varHandle = MethodHandles.insertCoordinates(varHandle, 1, 0);
     return (long) varHandle.get(memorySegment);
 }
```

### 获取嵌套结构体的B字段的值

```java
public static long getSubStructFieldB(MemorySegment memorySegment) {
    VarHandle varHandle = COMPLEX_LAYOUT.varHandle(
            MemoryLayout.PathElement.groupElement("sub_struct"),
            MemoryLayout.PathElement.groupElement("b")
    );
    varHandle = MethodHandles.insertCoordinates(varHandle, 1, 0);
    return (long) varHandle.get(memorySegment);
}
```

### 通过指针获取指针指向数组的第五个元素

这里要使用dereferenceElement这个特殊的Path元素

```java
public static long getLongArrayPtrIndex4(MemorySegment segment) {
    VarHandle varHandle = COMPLEX_LAYOUT.varHandle(
            MemoryLayout.PathElement.groupElement("long_array_ptr"),
            MemoryLayout.PathElement.dereferenceElement(),
            MemoryLayout.PathElement.sequenceElement(4)
    );
    //类似于
    //struct.long_array_ptr[4]
    varHandle = MethodHandles.insertCoordinates(varHandle, 1, 0);
    return (long) varHandle.get(segment);
}
```

### 获取某个结构体指针的指向的结构体的字段值

在此之前我们来回看一下上面的结构体声明，这里在声明long_and_long_ptr字段的的时候特意为这个address指定了withTargetLayout这个布局，这是我们能解指针的关键

```java
ValueLayout.ADDRESS.withTargetLayout(LONG_AND_LONG_LAYOUT).withName("long_and_long_ptr")
```

那么这个功能实现也很简单了

```java
 public static long getLongAndLongPtrFieldB(MemorySegment segment) {
     VarHandle varHandle = COMPLEX_LAYOUT.varHandle(
             MemoryLayout.PathElement.groupElement("long_and_long_ptr"),
             MemoryLayout.PathElement.dereferenceElement(),
             MemoryLayout.PathElement.groupElement("b")
     );
     //类似于
     //struct.long_and_long_ptr->b
     varHandle = MethodHandles.insertCoordinates(varHandle, 1, 0);
     return (long) varHandle.get(segment);
 }
```

### 自动布局

虽然JDK并没有提供对应的自动布局API但是我们可以自己写一个

```java
 public static StructLayout calAlignLayout(MemoryLayout... memoryLayouts) {
     long size = 0;
     long align = 1;
     ArrayList<MemoryLayout> layouts = new ArrayList<>();
     for (MemoryLayout memoryLayout : memoryLayouts) {
         //当前布局是否与size对齐
         if (size % memoryLayout.byteAlignment() == 0) {
             size = Math.addExact(size, memoryLayout.byteSize());
             align = Math.max(align, memoryLayout.byteAlignment());
             layouts.add(memoryLayout);
             continue;
         }
         long multiple = size / memoryLayout.byteAlignment();
         //计算填充
         long padding = (multiple + 1) * memoryLayout.byteAlignment() - size;
         size = Math.addExact(size, padding);
         //添加填充
         layouts.add(MemoryLayout.paddingLayout(padding));
         //添加当前布局
         layouts.add(memoryLayout);
         size = Math.addExact(size, memoryLayout.byteSize());
         align = Math.max(align, memoryLayout.byteAlignment());
     }
     //尾部对齐
     if (size % align != 0) {
         long multiple = size / align;
         long padding = (multiple + 1) * align - size;
         size = Math.addExact(size, padding);
         layouts.add(MemoryLayout.paddingLayout(padding));
     }
     return MemoryLayout.structLayout(layouts.toArray(MemoryLayout[]::new));
 }
```

