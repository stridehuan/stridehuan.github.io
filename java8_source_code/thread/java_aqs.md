# Java并发的核心 AQS
## 什么是AQS
AQS就是JDK的 java.util.concurrent.locks.AbstractQueuedSynchronizer 抽象类

参见官方文档 [AbstractQueuedSynchronizer](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/locks/AbstractQueuedSynchronizer.html)
> Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues.

它是一个基于先进先出等待队列的框架，用于实现阻塞所以及相关的同步器。ReentrantLock 和 CountDownLatch 都用到了基于AQS的同步器。

## AQS的使用
AQS的用法一般是定义一个同步器类，继承AQS抽象类，重写它的几个关键方法。
官方文档的说明是：
> To use this class as the basis of a synchronizer, redefine the following methods, as applicable, by inspecting and/or modifying the synchronization state using getState(), setState(int) and/or compareAndSetState(int, int):
>> tryAcquire(int)
>> tryRelease(int)
>> tryAcquireShared(int)
>> tryReleaseShared(int)
>> isHeldExclusively()

