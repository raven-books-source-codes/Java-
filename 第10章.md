[toc]

## CountDownlatch

### 为什么使用CountDownlatch而不是用join？

CountDownLatch与join方法的区别。一个区别是，调用一个子线程的join()方法后，该线程会一直被阻塞直到子线程运行完毕，而CountDownLatch则使用计数器来允许子线程运行完毕或者在运行中递减计数，也就是CountDownLatch可以在子线程运行的任何时候让await方法返回而不一定必须等到线程结束。另外，使用线程池来管理线程时一般都是直接添加Runable到线程池，这时候就没有办法再调用线程的join方法了，就是说countDownLatch相比join方法让我们对线程同步有更灵活的控制。

![image-20200311161255525](https://i.loli.net/2020/03/11/jbhFJGXxdZ4lIkC.png)

CountDownLatch 是使用AQS 实现的。通过下面的构造函数，你会 发现，实际上是把计数器的值赋给了AQ S 的状态变量s tate ，也就是这里使用A QS 的状态 值来表示计数器值。

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
Sync(int count) {
	setState(count);
}
```
### 1. void await （）方法

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```
![image-20200311161551301](https://i.loli.net/2020/03/11/VDatbZyu9EJwzKC.png)

### 3. void countDown （）方法

```java
public void countDown() {
    sync.releaseShared(1);
}
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {		// 设置state - 1
            doReleaseShared();		// 释放await阻塞的线程
            return true;
        }
        return false;
    }
    
            protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c - 1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```
需要唤醒因 调用CountDownLatch 的await 方法而被阻塞的线程，具体是调用AQS 的doReleaseShared 方法来激活阻塞的线程。

### 总结：

CountDownLatch 是使用AQS 实现的。使用AQS 的状态变量来存放计数器的值。首先在初始化 CountDownLatch 时设置状态值（计数器值），当多个线程调用countdown 方法时实际是原 子性递减AQS 的状态值。当线程调用await 方法后当前线程会被放入AQS 的阻塞队列等待计数器为0 再返回。其他线程调用countdown 方法让计数器值递减l ，当计数器值变为 0 时， 当前线程还要调用AQS 的doReleaseShared 方法来激活由于调用await（） 方法而被阻塞的线程。

## 10.2 回环屏障CyclicBarrier 原理探究

CountDownLatch 的计数器是一次性的，也就是等到计数器值变为 0 后，再调用CountDownLatch 的await 和countdown 方法都会立刻返回，这就起不到线 程同步的效果了。所以为了满足计数器可以重置的需要。

下面举个例子，证明CylicBarrier的可重用性：

```java
private static volatile CyclicBarrier cycleBarrier = new CyclicBarrier(2);

public static void main(String[] args) {
    ExecutorService pool = Executors.newFixedThreadPool(2);
    pool.execute(() -> {
        try {
            System.out.println(Thread.currentThread() + " Phase 1");
            cycleBarrier.await();
            System.out.println(Thread.currentThread() + " Phase 2");
            cycleBarrier.await();
            System.out.println(Thread.currentThread() + " Phase 3");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    });

    pool.execute(() -> {
        try {
            System.out.println(Thread.currentThread() + " Phase 1");
            cycleBarrier.await();
            System.out.println(Thread.currentThread() + " Phase 2");
            cycleBarrier.await();
            System.out.println(Thread.currentThread() + " Phase 3");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    });

    pool.shutdown();
    System.out.println("main is over");
}
```

```java
main is over
Thread[pool-1-thread-2,5,main] Phase 1
Thread[pool-1-thread-1,5,main] Phase 1
Thread[pool-1-thread-1,5,main] Phase 2
Thread[pool-1-thread-2,5,main] Phase 2
Thread[pool-1-thread-2,5,main] Phase 3
Thread[pool-1-thread-1,5,main] Phase 3

Process finished with exit code 0

```

### 10.2.2 实现原理探究

先看类图：

![image-20200311171205122](https://i.loli.net/2020/03/11/wse6DnvQ9GYgjFp.png)

来看看它的构造方法：

```java
    /**
     * Number of parties still waiting. Counts down from parties to 0
     * on each generation.  It is reset to parties on each new
     * generation or when broken.
     */
    private int count;
    /** The number of parties */
    private final int parties;
public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```

parties用于CyclicBairrer的复用。

下面来看CyclicBarrier 中的几个重要的方法。

### 1. int await （）方法

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```
### 3. int dowait(boolean timed, long nanos ）方法

```java

    /**
     * Main barrier code, covering the various policies.
     */
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)		// broken掉
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {	// 中断？
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            if (index == 0) {  // tripped
                Runnable command = barrierCommand;
                if (command != null) {
                    try {
                        command.run();		// 激活所有阻塞线程前，先run一个command
                    } catch (Throwable ex) {
                        breakBarrier();
                        throw ex;
                    }
                }
                nextGeneration();		// 激活所有阻塞线程，重置generation等
                return 0;
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)		// 是否设置超时
                        trip.await();		// 挂起，释放lock
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);	// 超时挂起，释放lock
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {	// 被中断了。
                        breakBarrier();		// 重置
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

```

本节首先通过案例说明了CycleBarrier 与CountDownLatch 的不同在于，前者是可以 复用的，并且前者特别适合分段任务有序执行的场景。然后分析了CycleBaJTier ， 其通过 独占锁ReentrantLock 实现计数器原子性更新，并使用条件变量队列来实现线程同步。

## 10.3 信号量Semaphore 原理探究

S emaphore 信号量也是Java 中的一个同步器，与C ountDownLatch 和CycleBarrier 不 同的是，它内部的计数器是递增的，并且在一开始初始化S emaphore 时－可以指定一个初始 值，但是并不需要知道需要同步的线程个数，而是在需要同步的地方调用acquire 方法时 指定需要同步的线程个数。

![image-20200311173110097](https://i.loli.net/2020/03/11/Gf7pQM8qrXTV1In.png)

### 1. void acquire （）方法

当前线程调用该方法的目的是希望获取一个信号量资源。如果当前信号量个数大于o, 则当前信号量的计数会减1 ， 然后该方法直接返回。否则如果当前信号量个数等于0 ，则 当前线程会被放入AQS 的阻塞队列。

    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
        public final void acquireSharedInterruptibly(int arg)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            if (tryAcquireShared(arg) < 0)		// 检查
                doAcquireSharedInterruptibly(arg);	// 放入阻塞队列
        }
        
```java
// nofaire  
final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;	// 返回负数，所以将会被阻塞
        }
    }
```
```java
    protected int tryAcquireShared(int acquires) {
        for (;;) {
            if (hasQueuedPredecessors())		// 检查前驱节点
                return -1;
            int available = getState();
            int remaining = available - acquires;  // 减去acquires
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
```
### 5.void release （）方法

```java
public void release() {
    sync.releaseShared(1);
}

```
```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();		// 调用unpark释放被阻塞的线程
        return true;
    }
    return false;
}
```
```java
    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))		// 设置状态减1
                return true;
        }
    }
```
### 总结

本节首先通过案例介绍了Semaphore的使用方法，Semaphore完全可以达到CountDownLatch的效果，但是Semaphore的计数器是不可以自动重置的，不过通过变相地改变aquire方法的参数还是可以实现CycleBanier的功能的。然后介绍了Semaphore的源码实现，Semaphore也是使用AQS实现的，并且获取信号量时有公平策略和非公平策略之分。

