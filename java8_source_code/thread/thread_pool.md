# JDK线程池

## ThreadPoolExecutor
默认的线程池实现类
### 重要属性和方法说明
- 属性
   - [ctl属性](#ctl)
- 方法
   - [execute方法](#execute)
   - [addWorker方法](#addWorker)
- 内部类

#### ctl
``` java
/**
 * is an atomic integer packing two conceptual fields
 * workerCount, indicating the effective number of threads
 * runState,    indicating whether running, shutting down etc
 */
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```
根据注释，这个变量里面保存了两个信息，有效线程数（workerCount，保存在前3位）、运行状态（runState，保存在后29位）
##### ctl位操作相关属性和方法
Java中int类型占用4字节，也就是32位（第1个是符号位）
``` java
private static final int COUNT_BITS = Integer.SIZE - 3; // =29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1; // 最大线程数 00011111111111111111111111111111

// runState is stored in the high-order bits (线程池的运行状态，保存在高3位)
private static final int RUNNING    = -1 << COUNT_BITS; // 高3位=111
private static final int SHUTDOWN   =  0 << COUNT_BITS; // 高3位=000
private static final int STOP       =  1 << COUNT_BITS; // 高3位=001
private static final int TIDYING    =  2 << COUNT_BITS; // 高3位=010
private static final int TERMINATED =  3 << COUNT_BITS; // 高3位=011

// Packing and unpacking ctl （封装ctl的位操作方法）
private static int runStateOf(int c)     { return c & ~CAPACITY; } // 获得线程池状态
private static int workerCountOf(int c)  { return c & CAPACITY; } // 获得有效线程数
private static int ctlOf(int rs, int wc) { return rs | wc; } // 合并线程池状态和有效线程数
```

#### execute
java.util.concurrent.ThreadPoolExecutor.execute
作用：将一个Runnable对象丢到线程池去执行
核心代码列出了execute方法执行时可能走的3个步骤：
``` java
// 1. workerCount小于corePoolSize，说明可以直接为command创建一个worker
int c = ctl.get();
if (workerCountOf(c) < corePoolSize) {
    if (addWorker(command, true))
        return;
    c = ctl.get();
}
// 2. workerCount大于等于corePoolSize，或者上一步创建worker失败，需要将command放到workQueue
if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
        reject(command);
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
// 3. 如果未能将command放入workQueue（可能是队列满了，或者上一步遇到了其它异常），尝试为command创建额外的worker，如果创建额外的worker失败，抛弃command
else if (!addWorker(command, false))
    reject(command);
```

#### addWorker
ava.util.concurrent.ThreadPoolExecutor.addWorker
作用：尝试根据firstTask生成一个worker并运行worker中的线程对象
整个方法分两个阶段

第一阶段，尝试自增workerCount，确保workerCount自增成功，且workerCount不能超过workerCount的上限
``` java
retry:
for (;;) {
    int c = ctl.get();
    int rs = runStateOf(c);

    // Check if queue empty only if necessary.
    if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                    firstTask == null &&
                    ! workQueue.isEmpty()))
        return false;

    // 不断轮询尝试完成workerCount的自增，如果自增成功，跳出大循环，如果发现workerCount已经达到上限了，返回false，如果自增失败，继续当前小循环
    for (;;) {
        int wc = workerCountOf(c);
        if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
            return false;
        if (compareAndIncrementWorkerCount(c))
            break retry;
        c = ctl.get();  // Re-read ctl
        if (runStateOf(c) != rs)
            continue retry;
        // else CAS failed due to workerCount change; retry inner loop
    }
}
```

第二阶段，生成worker，将worker添加到workers中，启动worker中的线程
``` java
w = new Worker(firstTask);
final Thread t = w.thread;
if (t != null) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // Recheck while holding lock.
        // Back out on ThreadFactory failure or if
        // shut down before lock acquired.
        int rs = runStateOf(ctl.get());

        if (rs < SHUTDOWN ||
                (rs == SHUTDOWN && firstTask == null)) {
            if (t.isAlive()) // precheck that t is startable
                throw new IllegalThreadStateException();
            workers.add(w);
            int s = workers.size();
            if (s > largestPoolSize)
                largestPoolSize = s;
            workerAdded = true;
        }
    } finally {
        mainLock.unlock();
    }
    if (workerAdded) {
        t.start();
        workerStarted = true;
    }
}
```

#### Worker
继承关系
``` java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable
```
构造方法
``` java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```
重写的run方法：
``` java
public void run() {
    runWorker(this);
}
```










