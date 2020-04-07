[toc]

## 9.2 简介

ScheduledThreadPoolExecutor是一个可以在指定二定延迟时间后或者定时进行 任务调度执行的线程池。

Executors其实是个工具类，它提供了好多静态方法，可根据用户的选择返回不同的线程池实例。ScheduledThreadPoo!Executor继承了ThreadPoo!Executor并实现了ScheduledExecutorService接口。线程池队列是DelayedWorkQueue，其和DelayQueue类似，是一个延迟队列。

ScheduledFutureTask 是具有返回值的任务， 继承自FutureTask 。FutureTask 的内部有 一个变量state 用来表示任务的状态， 一开始状态为NEW ， 所有状态为

```java
private static final int NEW=0；//初始状态
private static final int COMPLETING=1；//执行中状态
private static final int NORMAL=2；//正常运行结束状态
private static final int EXCEPTIONAL=3；//运行中异常
private static final int CANCELLED=4；//任务被取消
private static final int INTERRUPTING=5；//任务正在被中断
private static fina1 int INTERRUPTED=6；//任务已经被中断
```

![image-20200311130203930](https://i.loli.net/2020/03/11/XadsJjBvMG5iHgq.png)

可能的任务状态转换路径为

- NEW-> COMPLETING-> NORMAL//初始状态->执行中->正常结束
- NEW-> COMPLETING-> EXCEPTIONAL//初始状态->执行中->执行异常
- NEW-> CANCELLED//初始状态->任务取消
- NEW-> INTERRUPTING-> INTERRUPTED//初始状态->被中断中->被中断

ScheduledFutureTask 内部还有一个变量period 用来表示任务的类型，任务类型如下：

- period=0, 说明当前任务是一次性的，执行完毕后就退出了。

- period 为负数，说明当前任务为fixed-delay 任务，是固定延迟的定时可重复执行任务。

- period 为正数，说明当前任务为fixed-rate 任务， 是固定频率的定时可重复执行任务。

```java
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue(), handler);
    }
```

可以看到阻塞队列使用的是DelayedWorkQueue.

## 9.3 原理剖析

### 9.3.1 schedule(Runnable command, long delay,TimeUnit unit ）方法

该方法的作用是提交一个延迟执行的任务， 任务从提交时间算起延迟单位为unit 的 delay 时间后开始执行。提交的任务不是周期性任务，任务只会执行一次

```java

    /**
     * @throws RejectedExecutionException {@inheritDoc}
     * @throws NullPointerException       {@inheritDoc}
     */
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<Void> t = decorateTask(command,	// 转换为
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit),
                                          sequencer.getAndIncrement()));
        delayedExecute(t);
        return t;
    }

```

把提交的command 任务转换为ScheduledFutureTask 。 ScheduledFutureTask 是具体放入延迟队列里面的东西。由于是延迟任务，所以 ScheduledFutureTask 实现了long getDelay(TimeUnit unit ）和int compareTo(Delayed other ） 方 法。triggerTime 方法将延迟时间转换为绝对时间，也就是把当前时间的纳秒数加上延迟的 纳秒数后的long 型值,

```java
    ScheduledFutureTask(Runnable r, V result, long triggerTime,
                        long sequenceNumber) {
        super(r, result);
        this.time = triggerTime;
        this.period = 0;		// 设置只被调用一次
        this.sequenceNumber = sequenceNumber;
    }

    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }

    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
```
队列组织堆，使得队首总是要过期的任务，用于组织堆时的比较的方法：

```java
    public long getDelay(TimeUnit unit) {
        return unit.convert(time - System.nanoTime(), NANOSECONDS);
    }

    public int compareTo(Delayed other) {
        if (other == this) // compare zero if same object
            return 0;
        if (other instanceof ScheduledFutureTask) {
            ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
            long diff = time - x.time;
            if (diff < 0)
                return -1;
            else if (diff > 0)
                return 1;
            else if (sequenceNumber < x.sequenceNumber)
                return -1;
            else
                return 1;
        }
        long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
        return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
    }
```
再来看看如何将任务放到任务队列中：

```java
/**
 * Main execution method for delayed or periodic tasks.  If pool
 * is shut down, rejects the task. Otherwise adds task to queue
 * and starts a thread, if necessary, to run it.  (We cannot
 * prestart the thread to run the task because the task (probably)
 * shouldn't be run yet.)  If the pool is shut down while the task
 * is being added, cancel and remove it if required by state and
 * run-after-shutdown parameters.
 *
 * @param task the task
 */
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
        super.getQueue().add(task);
        if (!canRunInCurrentRunState(task) && remove(task))		// recheck
            task.cancel(false);
        else
            ensurePrestart();			// 保证最少有一个线程在运行
    }
}

    /**
     * Same as prestartCoreThread except arranges that at least one
     * thread is started even if corePoolSize is 0.
     */
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
    }

```
下面再来看看是如何运行该线程的：

```java
        public void run() {
            if (!canRunInCurrentRunState(this))
                cancel(false);
            else if (!isPeriodic())
                super.run();		// 如果不是周期性的，就会调用到这个方法
            else if (super.runAndReset()) {
                setNextRunTime();
                reExecutePeriodic(outerTask);
            }
        }

// 调用super.run()后，就调用下面这个函数
    public void run() {
        if (state != NEW ||		// 检查线程池状态
            !RUNNER.compareAndSet(this, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
// 真正的run在这里。
public T call() {
    task.run();
    return result;
}
```

run了以后，就需要设置线程的状态以及future的返回值：

```java
protected void set(V v) {
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        outcome = v;
        STATE.setRelease(this, NORMAL); // final state , 结束当前线程
        finishCompletion();
    }
}
```
exception同理。

### 9.3.2 scheduleWithFixedDelay(Runnable command,long initialDelay, long delay,TimeUnit unit）方法

该方法的作用是，当任务执行完毕后，让其延迟固定时间后再次运行（ fixed-delay 任 务）。其中initia!Delay 表示提交任务后延迟多少时间开始执行任务command, de lay 表示 当任务执行完毕后延长多少时间后再次运行command 任务， unit 是initia!Delay 和delay 的 时间单位。任务会一直重复运行直到任务运行中抛出了异常，被取消了，或者关闭了线程 池。

```java

    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0L)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          -unit.toNanos(delay),	//传递给ScheduledFutureTask 的period 变量的 值为－d elay , period<O 说明该任务为可重复执行的任务
                                          sequencer.getAndIncrement());
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

```

接着前文的run方法，此时isPeriodic返回true，所以执行super.runAndReset()方法：

```java
protected boolean runAndReset() {
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return false;
    boolean ran = false;
    int s = state;
    try {
        Callable<V> c = callable;
        if (c != null && s == NEW) {
            try {
                c.call(); // don't set result
                ran = true;
            } catch (Throwable ex) {
                setException(ex);
            }
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
    return ran && s == NEW;
}
```
这段代码和run方法类似，只是不会把任务状态设置NORMAL。而是重置为NEW。返回时需要检测是否ran以及重置状态成功：

```java
            else if (super.runAndReset()) {
                setNextRunTime();
                reExecutePeriodic(outerTask);
            }

        private void setNextRunTime() {
            long p = period;
            if (p > 0)		// fix-rate方法
                time += p;
            else		// fix-delay方法
                time = triggerTime(-p);	
        }

    /**
     * Requeues a periodic task unless current run state precludes it.
     * Same idea as delayedExecute except drops task rather than rejecting.
     *
     * @param task the task
     */
    void reExecutePeriodic(RunnableScheduledFuture<?> task) {	
        if (canRunInCurrentRunState(task)) {
            super.getQueue().add(task);		// 重新放入任务队列
            if (canRunInCurrentRunState(task) || !remove(task)) {
                ensurePrestart();
                return;
            }
        }
        task.cancel(false);
    }
```

成功后就会setNextRunTime。

#### 总结：

本节介绍的fixed”delay 类型的任务的执行原理为，当添加一个任务到延迟队列 后，等待initia!Delay 时－间，任务就会过期，过期的任务就会被从队列移除，并执行。执 行完毕后，会重新设置任务的延迟时间，然后再把任务放入延迟队列，循环往复。需要注 意的是，如果一个任务在执行中抛出了异常，那么这个任务就结束了，但是不影响其他任 务的执行。

### 9.3.3 scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit）方法

该方法相对起始时间点以固定频率调用指定的任务（ fixed-rate 任务） 。当把任务提交 到线程池并延迟initia!Delay 时间（ 时间单位为unit ）后开始执行任务command 。然后从 initia!Delay+period 时间点再次执行，而后在initia!Delay + 2 * period 时间点再次执行。

和AtFixedDelay的区别：

```java
        private void setNextRunTime() {
            long p = period;
            if (p > 0)		
                time += p;		// 固定间隔	
            else
                time = triggerTime(-p);		// triggerTime基于当前时间
        }
```

相对于fixed-delay 任务来说， fixed-rate 方式执行规则为，时间为initdelday + n叩eriod 时启动任务，但是如果当前任务还没有执行完，下一次要执行任务的时间到了， 则不会并发执行，下次要执行的任务会延迟执行，要等到当前任务执行完毕后再执行。



![](https://i.loli.net/2020/03/11/Qeru2gXSKOcN8FI.png)

## 总结：

![image-20200311153550456](https://i.loli.net/2020/03/11/pSuzlIROCHnyTLZ.png)