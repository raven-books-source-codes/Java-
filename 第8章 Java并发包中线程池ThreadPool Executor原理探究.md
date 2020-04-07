---

typora-copy-images-to: upload
---

[TOC]

线程池主要解决两个问题：一是当执行大量异步任务时线程池能够提供较好的性能。在不使用线程池时，每当需要执行异步任务时直接new一个线程来运行，而线程的创建和销毁是需要开销的。线程池里面的线程是可复用的，不需要每次执行异步任务时都重新创建和销毁线程。二是线程i也提供了一种资源限制和管理的手段，比如可以限制线程的个数，动态新增线程等。



![](https://i.loli.net/2020/03/10/5vgVP8rki3uATmG.jpg)

## 2. 1类图

![](https://i.loli.net/2020/03/10/uAP6pasXJZHMm73.jpg)

## 2.2 线程池简介

ThreadPoo!Executor继承了AbstractExecutorService，成员变量ct!是一个Integer的原子变量，用来记录线程池状态和线程池中线程个数，类似于ReentrantReadWriteLock使用一个变量来保存两种信息。

这里假设Integer 类型是32 位二进制表示，则其中高3 位用来表示线程池状态，后面29 位用来记录线程池线程个数。

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~COUNT_MASK; }
    private static int workerCountOf(int c)  { return c & COUNT_MASK; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

线程池状态含义如下：

- RUNNING ： 接受新任务并且处理阻塞队列里的任务。
- SHUTDOWN ：拒绝新任务但是处理阻塞队列里的任务。
- STOP ：拒绝新任务并且抛弃阻塞队列里的任务，同时会中断正在处理的任务。
- TIDYING ： 所有任务都执行完（包含阻塞队列里面的任务）后当前线程池活动线程数为0 ， 将要调用terminated 方法。

- TERMINATED ： 终止状态。terminated 方法调用完成以后的状态。

线程池状态转换列举如下：

- RUNNING -> SHUTDOWN ： 显式调用shutdown （） 方法， 或者隐式调用了finalize()，方法里面的shutdown（） 方法。

- RUNNING 或SHUTDOWN)-> STOP ： 显式调用shutdownNow（） 方法时。
- SHUTDOWN -> TIDYING ： 当线程池和任务队列都为空时。
- STOP -> TIDYING ： 当线程池为空时。
- TIDYING-> TERMINATED ： 当terminated() hook 方法执行完成时。

![](https://i.loli.net/2020/03/11/eNaMyvIlqUXwFRf.jpg)



线程池参数如下。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue)
```

•corePoolSize：线程池核心线程个数。

•workQueue：用于保存等待执行的任务的阻塞队列，比如基于数组的有界ArrayBlockingQueue、基于链表的无界LinkedBlockingQueue、最多只有一个元素的同步队列SynchronousQueue及优先级队列PriorityBlockingQueue等。

•maximunPoolSize：线程池最大线程数量。 

•ThreadFactory：创建线程的工厂。

•RejectedExecutionHandler：饱和策略，当队列满并且线程个数达到maximunPoolSize后采取的策略，比如AbortPolicy（抛出异常〉、CallerRunsPolicy（使用调用者所在线程来运行任务）、DiscardOldestPolicy（调用poll丢弃一个任务，执行当前任务）及DiscardPolicy（默默丢弃，不抛出异常〉

•keeyAliveTime：存活时间。如果当前线程池中的线程数量比核心线程数量多，并且是闲置状态，则这些闲置的线程能存活的最大时间。

•TimeUnit：存活时间的时间单位。

线程池类型如下。

- newFixedThreadPool：创建一个核心线程个数和最大线程个数都为nThreads的线程池，并且阻塞队列长度为Integer.MAXVALUE.keeyLivetIme=0说明只要线程个

```java
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

- newSingleThreadExecutor：创建一个核心线程个数和最大线程个数都为1的线程池，并且阻塞队列长度为Integer.MAX_VALUE。keeyAliveTime=O说明只要线程个数比核心线程个数多并且当前空闲则回收。



```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
- newCachedThreadPool：创建一个按需创建线程的线程池，初始线程个数为0，最多线程个数为Integer.MAX_VALUE，并且阻塞队列为同步队列。keeyAliveTime=60说明只要当前线程在60s内空闲则回收。这个类型的特殊之处在于，加入同步队列的任务会被马上执行，同步队列里面最多只有一个任务。

```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());	// 用的同步队列
    }
```

## 8.3 源码分析

### 8.3.1 public void execute(Runnable command)

execute 方法的作用是提交任务command 到线程池进行执行。用户线程提交任务到线 程池的模型图如图8-2 所示。

![image-20200311104134530](https://i.loli.net/2020/03/11/oyLV56bhZTIJr2R.png)

ThreadPoo!Executor 的实现实际是一个生产消费模型。

```java

    /**
     * Executes the given task sometime in the future.  The task
     * may execute in a new thread or in an existing pooled thread.
     *
     * If the task cannot be submitted for execution, either because this
     * executor has been shutdown or because its capacity has been reached,
     * the task is handled by the current {@link RejectedExecutionHandler}.
     *
     * @param command the task to execute
     * @throws RejectedExecutionException at discretion of
     *         {@code RejectedExecutionHandler}, if the task
     *         cannot be accepted for execution
     * @throws NullPointerException if {@code command} is null
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)		// 执行到这里，其余线程就不会设置thradpool的state为shutdown了吗？毕竟没有加锁，或者cas操作。
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

```

重点分三步：

- 小于coresize，则添加worker，
- 判定线程的执行状态，非Running则不接收新线程。这里执行了recheck
- 继续尝试添加新线程。

重点在addWorker方法上：

```java

    /*
     * Methods for creating, running and cleaning up after workers
     */

    /**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (int c = ctl.get();;) {
            // Check if queue empty only if necessary.
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP)
                    || firstTask != null
                    || workQueue.isEmpty()))
                return false;

            for (;;) {
                if (workerCountOf(c)
                    >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateAtLeast(c, SHUTDOWN))
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        /**
        * 上半部分是增加thread的count数
        * 下半部分是分发work
        */
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();

                    if (isRunning(c) ||
                        (runStateLessThan(c, STOP) && firstTask == null)) {
                        if (t.getState() != Thread.State.NEW)
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        workerAdded = true;
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

```

第一部分双重循环的目的是通过CAS 操作增加线程数： 第二部分主要是把并发安全的任务添加到workers 里面。

### 8.3.2 工作线程Worker 的执行

由addWorker中的t.start()转到：

```java
        /** Delegates main run loop to outer runWorker. */
        public void run() {
            runWorker(this);
        }

```

```java

    /**
     * Main worker run loop.  Repeatedly gets tasks from queue and
     * executes them, while coping with a number of issues:
     *
     * 1. We may start out with an initial task, in which case we
     * don't need to get the first one. Otherwise, as long as pool is
     * running, we get tasks from getTask. If it returns null then the
     * worker exits due to changed pool state or configuration
     * parameters.  Other exits result from exception throws in
     * external code, in which case completedAbruptly holds, which
     * usually leads processWorkerExit to replace this thread.
     *
     * 2. Before running any task, the lock is acquired to prevent
     * other pool interrupts while the task is executing, and then we
     * ensure that unless pool is stopping, this thread does not have
     * its interrupt set.
     *
     * 3. Each task run is preceded by a call to beforeExecute, which
     * might throw an exception, in which case we cause thread to die
     * (breaking loop with completedAbruptly true) without processing
     * the task.
     *
     * 4. Assuming beforeExecute completes normally, we run the task,
     * gathering any of its thrown exceptions to send to afterExecute.
     * We separately handle RuntimeException, Error (both of which the
     * specs guarantee that we trap) and arbitrary Throwables.
     * Because we cannot rethrow Throwables within Runnable.run, we
     * wrap them within Errors on the way out (to the thread's
     * UncaughtExceptionHandler).  Any thrown exception also
     * conservatively causes thread to die.
     *
     * 5. After task.run completes, we call afterExecute, which may
     * also throw an exception, which will also cause thread to
     * die. According to JLS Sec 14.20, this exception is the one that
     * will be in effect even if task.run throws.
     *
     * The net effect of the exception mechanics is that afterExecute
     * and the thread's UncaughtExceptionHandler have as accurate
     * information as we can provide about any problems encountered by
     * user code.
     *
     * @param w the worker
     */
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    try {
                        task.run();
                        afterExecute(task, null);
                    } catch (Throwable ex) {
                        afterExecute(task, ex);
                        throw ex;
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }

```

主要的就是getTask函数，从中获取task，要不就是超时返回null:

```java

    /**
     * Performs blocking or timed wait for a task, depending on
     * current configuration settings, or returns null if this worker
     * must exit because of any of:
     * 1. There are more than maximumPoolSize workers (due to
     *    a call to setMaximumPoolSize).
     * 2. The pool is stopped.
     * 3. The pool is shutdown and the queue is empty.
     * 4. This worker timed out waiting for a task, and timed-out
     *    workers are subject to termination (that is,
     *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
     *    both before and after the timed wait, and if the queue is
     *    non-empty, this worker is not the last thread in the pool.
     *
     * @return task, or null if the worker must exit, in which case
     *         workerCount is decremented
     */
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();

            // Check if queue empty only if necessary.
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

```

### 8.3.1-2总结

![image-20200311123113007](https://i.loli.net/2020/03/11/cID9wBAXjVKkrRt.png)

![image-20200311123524757](https://i.loli.net/2020/03/11/iHXqOS6mBdr19Ay.png)

总的来说就两个，如果worker数不够corepoolsize，就添加worker，提高并发量，够了就offer到workQueue中。

### 8.3.3 shutdown 操f乍

调用shutdown 方法后，线程池就不会再接受新的任务了，但是工作队列里面的任务 还是要执行的

```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();		// 检查当前线程是否有权限shutdown
            advanceRunState(SHUTDOWN);		// 设置当前threadpool为shutdown
            interruptIdleWorkers();		// 中断那些在等待队列中的thread
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }


    /**
     * Interrupts threads that might be waiting for tasks (as
     * indicated by not being locked) so they can check for
     * termination or configuration changes. Ignores
     * SecurityExceptions (in which case some threads may remain
     * uninterrupted).
     *
     * @param onlyOne If true, interrupt at most one worker. This is
     * called only from tryTerminate when termination is otherwise
     * enabled but there are still other workers.  In this case, at
     * most one waiting worker is interrupted to propagate shutdown
     * signals in case all threads are currently waiting.
     * Interrupting any arbitrary thread ensures that newly arriving
     * workers since shutdown began will also eventually exit.
     * To guarantee eventual termination, it suffices to always
     * interrupt only one idle worker, but shutdown() interrupts all
     * idle workers so that redundant workers exit promptly, not
     * waiting for a straggler task to finish.
     */
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }


    /**
     * Transitions to TERMINATED state if either (SHUTDOWN and pool
     * and queue empty) or (STOP and pool empty).  If otherwise
     * eligible to terminate but workerCount is nonzero, interrupts an
     * idle worker to ensure that shutdown signals propagate. This
     * method must be called following any action that might make
     * termination possible -- reducing worker count or removing tasks
     * from the queue during shutdown. The method is non-private to
     * allow access from ScheduledThreadPoolExecutor.
     */
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateLessThan(c, STOP) && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }

```

在如上代码中， 首先使用CAS 设置当前线程池状态为TIDYING ，如果设置成功则 执行扩展接口terminated 在线程池状态变为TERMINATED 前做一些事情，然后设置当 前线程池状态为TERMINATED 。最后调用termination . signalAll （） 激活因调用条件变量 termination 的await 系列方法而被阻塞的所有线程，关于这一点随后讲到awaitTermination 方法时具体讲解。

### 8.3.4 shutdownNow 操作

调用shutdownNow 方法后， 线程池就不会再接受新的任务了，并且会丢弃工作队列 里面的任务， 正在执行的任务会被中断， 该方法会立刻返回，并不等待激活的任务执行完 成。返回值为这时候队列里面被丢弃的任务列表。

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        interruptWorkers();
        tasks = drainQueue();		//将当前任务队列里面的任务移动到tasks 列表。
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```
中带你在中断workers：

```java
    private void interruptWorkers() {
        // assert mainLock.isHeldByCurrentThread();
        for (Worker w : workers)
            w.interruptIfStarted();
    }
```

### 8.3.5 awaitTermination

当线程调用awaitTermination 方法后，当前线程会被阻塞，直到线程池状态变为 TERMINATED 才返回， 或者等待时间超时才返回

```java

    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            while (runStateLessThan(ctl.get(), TERMINATED)) {
                if (nanos <= 0L)
                    return false;
                nanos = termination.awaitNanos(nanos);
            }
            return true;
        } finally {
            mainLock.unlock();
        }
    }

```

- 在讲解shutdown方法时提到过，当线程池状态变为TERMINATED时，会调用termination.signa!All（）用来激活调用条件变量termination的await系列方法被阻塞的所有线程，所以如果在调用awaitTem1ination之后又调用了shutdown方法，并且在shutdown内部将线程池状态设置为TERMINATED，则termination.awaitNanos方法会返回。

- 另外在工作线程Worker的runWorker方法内，当工作线程运行结束后，会调用processWorkerExit方法，在processWorkerExit方法内部也会调用tryTerminate方法测试当前是否应该把线程池状态设置 TERMINATED，如果是，则也会调用termination.signa!All（）用来激活调用线程池的awaitTermination方法而被阻塞的线程。而且当等待时间超时后，te1mination.awaitNanos也会返回，这时候会重新检查当前线程池状态是否为TERMINATED，如果是则直接返回，否则继续阻塞挂起自己。

## 总结：

线程池巧妙地使用一个Integer 类型的原子变量来记录线程池状态和线程池中的钱程 个数。通过线程池状态来控制任务的执行，每个Worker 线程可以处理多个任务。线程池 通过线程的复用减少了线程创建和销毁的开销。