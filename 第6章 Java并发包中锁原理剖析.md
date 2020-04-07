[TOC]

## 6.1 LockSupport 工具类

### 1. void park(）方法

如果调用park方法的线程已经拿到了与LockSupport关联的许可证，则调用Locksupport.park（）时会马上返回，否则调用线程会被禁止参与线程的调度，也就是会被阻塞挂起。

需要注意的是，因调用park（）方法而被阻塞的线程被其他线程中断而返回时并不会抛出InterruptedException异常。

### 2. void unpark()方法

LockSupoort类采用的是发授“许可证”的方式。unpark即是给予一个线程许可证，只当当一个线程得到许可后调用park方法才不会被阻塞。

### 4. park(Object blocker）方法

可以使用诊断器查看阻塞对象。

`jstack pid` 命令查看线程堆枝时可以看到如下输出结果。

## 6.2 抽象同步队列AQS 概述

### 6.2.1 AQS 一一锁的底层支持

![](https://pic.downk.cc/item/5e64428898271cb2b836a24b.jpg)

AQS是一个FIFO的双向队列，其内部通过节点head和tail记录队首和队尾元素，队列元素的类型为Node。其中Node中的thread变量用来存放进入AQS队列里面的线程：

在AQS中维持了一个单一的状态信息state，可以通过getState、setState、compareAndSetState函数修改其值。对于ReentrantLock的实现来说，state可以用来表示当前线程获取锁的可重入次数；对于读写锁ReentrantReadWriteLock来说，state的高16位表示读状态，也就是获取该读锁的次数，低16位表示获取到写锁的线程的可重入次数；对于semaphore来说，state用来表示当前可用信号的个数：对于CountDownlatch来说，state用来表示计数器当前的值。

AQS有个内部类ConditionObject，用来结合锁实现线程同步。ConditionObject可以直接访问AQS对象内部的变量，比如state状态值和AQS队列。ConditionObject是条件变量，每个条件变量对应一个条件队列（单向链表队列），其用来存放调用条件变量的await方法后被阻塞的线程，如类图所示，这个条件队列的头、尾元素分别为自rstWaiter和lastWaiter。

**线程同步的关键是对状态值state 进行操作。根据state 是否属于一个线程，操作state 的方式分为独占方式和共享方式**:

- 独占方式获取的资源是与具体线程绑定的
- 共享方式的资源与具体线程是不相关的

>  锁的独占与共享
>
> ​      java并发包提供的加锁模式分为独占锁和共享锁，独占锁模式下，每次只能有一个线程能持有锁，ReentrantLock就是以独占方式实现的互斥锁。共享锁，则允许多个线程同时获取锁，并发访问 共享资源，如：ReadWriteLock。AQS的内部类Node定义了两个常量SHARED和EXCLUSIVE，他们分别标识 AQS队列中等待线程的锁获取模式。
>
> ​     很显然，独占锁是一种悲观保守的加锁策略，它避免了读/读冲突，如果某个只读线程获取锁，则其他读线程都只能等待，这种情况下就限制了不必要的并发性，因为读操作并不会影响数据的一致性。共享锁则是一种乐观锁，它放宽了加锁策略，允许多个执行读操作的线程同时访问共享资源。 java的并发包中提供了ReadWriteLock，读-写锁。它允许一个资源可以被多个读操作访问，或者被一个 写操作访问，但两者不能同时进行。

#### **独占锁流程：**

获取阶段：当一个线程调用acquire(intarg）方法获取独占资源时，会首先使用tryAcquire方法尝试获取资源，具体是设置状态变量state的值，成功则直接返回，失败则将当前线程封装为类型为Node.EXCLUSIVE的Node节点后插入到AQS阻塞队列的尾部，并调用LockSupport.park(this）方法挂起自己。

```java
    public final void acquire(long arg) {
        if (!tryAcquire(arg) &&		// 使用tryAcquire设置state变量（cas方式）
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 不成功的话就设置Node为独占，并添加到阻塞队列中
            selfInterrupt(); // 中断？ 书上说的是调用park(this)阻塞，但是查源码是interrupt
    }
```

释放阶段：当一个线程调用release(intarg）方法时会尝试使用tryRelease操作释放资源，这里是设置状态变量state的值，然后调用LockSupport.unpark(thread）方法激活AQS队列里面被阻塞的一个线程（thread）。被激活的线程则使用tryAcquire尝试，看当前状态变量state的值是否能满足自己的需要，满足则该线程被激活，然后继续向下运行，否则还是会被放入AQS队列并被挂起。

```java
    public final boolean release(long arg) {
        if (tryRelease(arg)) {	// 设置state
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);	// 释放一个等待队列中的node
            return true;
        }
        return false;
    }
```

AQS类并没有提供可用的句Acquire和町Release方法，正如AQS是锁阻塞和同步器的基础框架一样，t可Acquire和tryRelease需要由具体的子类来实现。子类在实现tryAcquire和tryRelease时要根据具体场景使用CAS算法尝试修改state状态值，成功则返回true，否则返回false。子类还需要定义，在调用acquire和release方法时state状态值的增减代表什么含义。

以ReentrantLock为例，accquire时需要设置state为1，并关联锁为当前线程，代码如下：

```java
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {		// 设置state
                    setExclusiveOwnerThread(current);		// 设置独占锁与当前线程关联
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;	
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {	
                free = true;
                setExclusiveOwnerThread(null);		// 解除独占锁
            }
            setState(c);
            return free;
        }
```

#### 共享锁流程：

获取:当线程调用acquireShared(intarg）获取共享资源时，会首先使用trγAcq山reShared尝试获取资源，具体是设置状态变量state的值，成功则直接返回，失败则将当前线程封装为类型为Node.SHARED的Node节点后插入到AQS阻塞队列的尾部，并使用LockSupport.park(this）方法挂起自己。

```java
    public final void acquireShared(long arg) {
        if (tryAcquireShared(arg) < 0)		// 尝试获取
            doAcquireShared(arg);		// 获取失败，做acquires需求操作,内部有park操作，挂起
    }
```

释放：当一个线程调用releaseShared(inta电）时会尝试使用tryReleaseShared操作释放资源，这里是设置状态变量state的值，然后使用LockSupport.unpark(thread）激活AQS队列里面被阻塞的一个线程（thread）。被激活的线程则使用tryReleaseShared查看当前状态变量state的值是否能满足自己的需要，满足则该线程被撤活，然后继续向下运行，否则还是会被放入AQS队列并被挂起。

```java
    public final boolean releaseShared(long arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

ReentrantReadWriteLock就是共享锁。

**基于AQS实现的锁除了需要重写上面介绍的方法外，还需要重写isHeldExclusively方法，来判断锁是被当前线程独占还是被共享。**

#### AQS的入队操作

```java
    private Node enq(Node node) {
        for (;;) {
            Node oldTail = tail;
            if (oldTail != null) {
                node.setPrevRelaxed(oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return oldTail;
                }
            } else {
                initializeSyncQueue();
            }
        }
    }
```

![](https://pic.downk.cc/item/5e64697598271cb2b8486672.jpg)

### 6.2.2 AQS一一条件变量的支持

条件变量的await和singal就和基础篇中讲解的wait和notify类似。后者获得内置的监视锁，前置获得条件变量的锁。下面看一个例子：

````java
public class Test {
    private static ReentrantLock lock = new ReentrantLock();
    private static Condition condition = lock.newCondition();
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
            try {
                lock.lock();
                System.out.println(Thread.currentThread());
                condition.await();
                System.out.println("continue");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally{
                lock.unlock();
            }

        });
        Thread t2 = new Thread(()->{
            try{
                lock.lock();
                System.out.println(Thread.currentThread());
                condition.signal();
                System.out.println("signal");
            }finally{
                lock.unlock();
            }
        });
        t1.start();
        Thread.sleep(1000);
        t2.start();

        t1.join();
        t2.join();
        System.out.println("main is over");
    }
}
````

#### await实现

```java

  
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();	// 添加到条件队列中
            long savedState = fullyRelease(node);	// 释放当前锁
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);	// 将自己挂起
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

```

#### 再来看看signal

```
        public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;		// 从condition队列中取出第一个元素
            if (first != null)
                doSignal(first);		// 将它添加到AQS的队列中，等待调度
        }
```

## 这里是本章的重点 AQS队列和Condition队列的工作流程	



注意：当多个线程同时调用lock.lock（）方法获取锁时，只有一个线程获取到了锁，其他线程会被转换为Node节点插入到lock锁对应的AQS阻塞队列里面，并做自旋CAS尝试获取锁。

如果获取到锁的线程又调用了对应的条件变量的await（）方法，则该线程会释放获取到的锁，并被转换为Node节点插入到条件变量对应的条件队列里面。

这时候因为调用lock.lock（）方法被阻塞到AQS队列里面的一个线程会获取到被释放的锁，如果该线程也调用了条件变量的await（）方法则该线程也会被放入条件变量的条件队列里面。

当另外一个线程调用条件变量的signal（）或者signa!All（）方法时，会把条件队列里面的一个或者全部Node节点移动到AQS的阻塞队列里面，等待时机获取锁。

![](https://pic.downk.cc/item/5e644e2498271cb2b83c9428.jpg)

### 6.2.3 基于AQS 实现自定义同步器

主要是要重写tryRequire和tryRelease以及isHeildExclusive方法：

```java
    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            // 设置state 为1 ，绑定当前线程
            assert arg == 1 : "arg must be 1";
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            // 设置state为0，解绑线程
            setExclusiveOwnerThread(null);
            setState(0); // 这里不需要原子设置？
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return true;
        }

        public Condition newCondition() {
            return new ConditionObject();
        }
    }
        public void lock() {
        sync.acquire(1);
    }

    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    public void unlock() {
        sync.release(1);
    }

    public boolean isLocked() {
        return sync.isHeldExclusively();
    }

    public Condition newCondition() {
        return sync.newCondition();
    }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

```



## 6.3 独占锁Reentrant Lock 的原理

先看类图，可以看到ReentrantLock内部关联一个Sync，其底层实现还是用的AQS，并且支持公平/非公平sync。

![](https://pic.downk.cc/item/5e64679498271cb2b847dbb2.jpg)

源码比较简单:

### 6.3.1 获得锁

FaireSync:

```java
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
        
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {	// 初始化
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {	// 重入，检测移除
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;		// 其余线程则false
        }

```

NofaireSync:

```java

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&	// 这里的hasQueuedPredecessors就是公平调度的核心
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

    /**
     * Queries whether any threads have been waiting to acquire longer
     * than the current thread.
 	 */
    public final boolean hasQueuedPredecessors() {
        Node h, s;
        if ((h = head) != null) {
            if ((s = h.next) == null || s.waitStatus > 0) {
                s = null; // traverse in case of concurrent cancellation
                for (Node p = tail; p != h && p != null; p = p.prev) {
                    if (p.waitStatus <= 0)
                        s = p;
                }
            }
            if (s != null && s.thread != Thread.currentThread())	// 检测是否有下一节点，或者当前thread不是s
                return true;
        }
        return false;
    }


```

### 6.3.3 释放锁

```java

        @ReservedStackAccess
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

```

总结：

![](https://pic.downk.cc/item/5e647a2698271cb2b85068e4.jpg)

## 6.4 读写锁ReentrantReadWritelock 的原理

类图：

![](https://pic.downk.cc/item/5e646dc998271cb2b849bc53.jpg)

ReentrantReadWriteLock 巧妙地使用state 的高16 位表示读状态，也就是获取到读锁的次数；使用低16 位表示获取到写锁的线程的可重入次数。

### 6.4.2 写锁的获取与释放

**获取**

```java

        @ReservedStackAccess
        protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
        
        // FairSync
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        
        //NoFaire
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }

```

**释放**

```java
        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())		// 是否是当前线程
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;	
            boolean free = exclusiveCount(nextc) == 0;	// 	是否减到了0
            if (free)
                setExclusiveOwnerThread(null);	// 解绑
            setState(nextc);
            return free;
        }
```

### 6.4.3 读锁的获取与释放

获取：

```java

        @ReservedStackAccess
        protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null ||
                        rh.tid != LockSupport.getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)		// 这里没怎么看懂
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }

```

释放：

```java

        @ReservedStackAccess
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null ||
                    rh.tid != LockSupport.getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }

```

总结：

![](https://pic.downk.cc/item/5e647a0b98271cb2b85055fb.jpg)

## 6.5 JDK 8 中新增的Stamped Lock 锁探究

提供三种锁与对应的时间戳。

- 写锁
- 悲观读锁readLock
- 乐观读锁町OptimisticRead

![](https://pic.downk.cc/item/5e647ff398271cb2b8524c2c.jpg)