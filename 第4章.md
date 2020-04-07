[TOC]

## 4.1 原子变量操作类

AtomicLong是原子性递增或者递减类，其内部使用Unsafe 来实现，我们看下面的代码。

```java
public class AtomicLong extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 1927816293512124184L;

    /**
     * Records whether the underlying JVM supports lockless
     * compareAndSet for longs. While the intrinsic compareAndSetLong
     * method works in either case, some constructions should be
     * handled at Java level to avoid locking user-visible locks.
     */
    static final boolean VM_SUPPORTS_LONG_CAS = VMSupportsCS8();

    /**
     * Returns whether underlying JVM supports lockless CompareAndSet
     * for longs. Called only once and cached in VM_SUPPORTS_LONG_CAS.
     */
    private static native boolean VMSupportsCS8();

    /*
     * This class intended to be implemented using VarHandles, but there
     * are unresolved cyclic startup dependencies.
     */
    private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
    private static final long VALUE = U.objectFieldOffset(AtomicLong.class, "value");

    private volatile long value; //真正保存值得地方
    ...
}
```

看看进行操作加减设置的代码：

public final long incrementAndGet() {
        return U.getAndAddLong(this, VALUE, 1L) + 1L;
    }

```java

    public final void set(long newValue) {
        // See JDK-8180620: Clarify VarHandle mixed-access subtleties
        U.putLongVolatile(this, VALUE, newValue);
    }
    

    public final long incrementAndGet() {
        return U.getAndAddLong(this, VALUE, 1L) + 1L;
    }


    public final long decrementAndGet() {
        return U.getAndAddLong(this, VALUE, -1L) - 1L;
    }

```
可以看到底层都是用Unsafe类来实现的。

我们来看看个体AndAddLong是如何实现的：

```java
@HotSpotIntrinsicCandidate
public final long getAndAddLong(Object o, long offset, long delta) {
    long v;
    do {
        v = getLongVolatile(o, offset);
    } while (!weakCompareAndSetLong(o, offset, v, v + delta));
    return v;
}
```
可以看到是使用了CAS来进行操作的。

下面是AtomicLong的使用例子：

```java
    private static int[] arr1 = new int[]{12,213,4,523,16,75,758,0,0,0};
    private static int[] arr2 = new int[]{312,4,51,6,75,41,23,0,0,0,0};

    private static AtomicLong zeroNumbers = new AtomicLong();

    public static void main(String[] args) throws InterruptedException {
       // thread 1 to detect the arr1
       Thread t1 = new Thread(()->{
           for(int i : arr1){
               if(i == 0){
                   zeroNumbers.incrementAndGet();
               }
           }
       });
        Thread t2 = new Thread(()->{
            for(int i : arr2){
                if(i == 0){
                    zeroNumbers.incrementAndGet();
                }
            }
        });
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(zeroNumbers);
    }
```

在没有原子类的情况下，实现计数器需要使用一定的同步措施，比如使用synchronized关键字等，但是这些都是阻塞算法，对性能有一定损耗，而本章介绍的这些原子操作类都使用CAS非阻塞算法，性能更好。但是在高并发情况下AtomicLong还会存性能问题。JDK8提供了一个在高并发下性能更好的LongAdder.

## 4.2 LongAdder

LongAddr解决了AtomicLong这样一个问题：当多个线程争用同一个AtomicLong对象时，因为维护的是同一个long，所以cas造成多个线程阻塞。LongAddr将一个long增加为一个cell组，这样多个线程可以争用多个cell对象，避免自旋阻塞。

![](https://pic.downk.cc/item/5e6311a598271cb2b8a02d94.jpg)

![1583550909120](C:\Users\Raven\AppData\Roaming\Typora\typora-user-images\1583550909120.png)

同时cell类通过Contented注解解决了伪共享的问题。

### 4.2.2 LongAdder 代码分析

问题：（1)LongAdder的结构是怎样的？(2）当前线程应该访问Cell数组里面的哪一个Cell元素？(3)如何初始化Cell数组？C4)Cell数组如何扩容？(5）线程访问分配的Cell元素有冲突后如何处理？(6）如何保证线程操作被分配的Cell元素的原子性？

问题1，LongAdder的结构是怎样的？：LongAdder中维护的是Cell数组和一个base变量。

问题6，如何保证线程操作被分配的Cell元素的原子性？：Cell内部有volatile关键字修饰的value变量，且对该变量进行操作都是原子操作。通过Unsafe类完成。

问题2，当前线程应该访问Cell数组里面的哪一个Cell元素？：通过getProbe&m来决定

```
    public void add(long x) {
        Cell[] cs; long b, v; int m; Cell c;
        if ((cs = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (cs == null || (m = cs.length - 1) < 0 ||
                (c = cs[getProbe() & m]) == null ||		//问题2
                !(uncontended = c.cas(v = c.value, v + x))) 
                longAccumulate(x, null, uncontended);
        }
    }
```

问题3：如何初始化Cell数组？

```java
            else if (cellsBusy == 0 && cells == cs && casCellsBusy()) {
                try {                           // Initialize table
                    if (cells == cs) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        break done;
                    }
                } finally {
                    cellsBusy = 0;
                }
            }
```

cellsBusy用于判定当前cell数组是否正在被初始化或扩容。

问题4：Cell数组如何扩容?

```java
               else if (n >= NCPU || cells != cs)  
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == cs)        // Expand table unless stale
                            cells = Arrays.copyOf(cs, n << 1);
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
```

只有当cell数组的数量小于cpu个数，且发生了冲突的时候才进行扩容。

问题5：线程访问分配的Cell元素有冲突后如何处理？

```java
 h = advanceProbe(h);
```

对CAS 失败的线程重新计算当前线程的随机值threadLocaRandomProbe,以减少下次访问cells 元素时的冲突机会。

注意LongAdder的用法，最好用于求和等统计信息。

## 4.3 LongAccumulator 类原理探究

LongAdder是LongAccumulator的一个特例，看看LongAccmulator的构造函数：

```java
    public LongAccumulator(LongBinaryOperator accumulatorFunction,
                           long identity) {
        this.function = accumulatorFunction;
        base = this.identity = identity;
    }

publiC interface LongBinaryOperator {
／／根据两个参数计算并返回一个值
	long applyAsLong ( long left , long right ) ;
}
```

LongAdder的区别在于

```

    /**
     * Updates with the given value.
     *
     * @param x the value
     */
    public void accumulate(long x) {
        Cell[] cs; long b, v, r; int m; Cell c;
        if ((cs = cells) != null
            || ((r = function.applyAsLong(b = base, x)) != b
                && !casBase(b, r))) {	// r = function.applyAsLong,但是在LongAdder类中是直接b+x
            boolean uncontended = true;
            if (cs == null
                || (m = cs.length - 1) < 0
                || (c = cs[getProbe() & m]) == null
                || !(uncontended =
                     (r = function.applyAsLong(v = c.value, x)) == v
                     || c.cas(v, r)))
                longAccumulate(x, function, uncontended);
        }
    }

```

可自定义function，即一个双目运算符。