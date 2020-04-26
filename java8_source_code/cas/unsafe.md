# CAS底层Unsafe相关操作分析

以 AtomicInteger 类为例。

## 属性

``` java
// Unsafe 单例
private static final Unsafe unsafe = Unsafe.getUnsafe();
// value 属性内存偏移量
private static final long valueOffset;
// 线程可见的 value 属性，用于保存属性值
private volatile int value;
```

## 对象初始化

静态代码块：

``` java
static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}
```

通过 unsafe 单例获得 value 属性的偏移量，一旦类确定后，属性的偏移量就确定了，所以放在静态代码块中实现。

## compareAndSet 方法

代码实现：

``` java
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

其中 Unsafe 的 compareAndSwapInt 方法是一个 native 方法。

## compareAndSwapInt 方法

该方法的前两个属性可以确定 value 属性的内存地址，两个属性分别 预期值 expect，更新值 update。

本质上是通过 c 语言调用 cpu 底层指令实现的。

底层基于 cpu 锁实现原子操作，主要基于两种 cpu 锁：

1. 总线锁（相当于阻断了其它核心对内存的访问，但开销比较大）
2. 缓存锁（锁住内存区域的缓存行，开销相对小）

关于unsafe处理器层面的源码分析参考了这篇博文：[并发编程之CAS（Compare and Swap）原理Unsafe类](https://cloud.tencent.com/developer/article/1189884)



[上一页](../index.md)