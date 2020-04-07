[TOC]

## 3.1 Random 类及其局限性

先看下random是如何产生随机数的：

![](https://pic.downk.cc/item/5e6300ab98271cb2b89b2e07.jpg)

1. 由此可见，新的随机数的生成需要两个步骤：
   首先根据老的种子生成新的种子。
   然后根据新的种子来计算新的随机数。

在多线程下多个线程可能都拿同一个老的种子去执行步骤（4）以计算新的种子，这会导致多个线程产生的新种子是一样的，由于步骤（5）的算法是固定的，所以会导致多个线程产生相同的随机值，这并不是我们想要的。

所以步骤（4）要保证原子性，也就是说当多个线程根据同一个老种子计算新种子时，第一个线程的新种子被计算出来后，第二个线程要丢弃自己老的种子，而使用第一个线程的新种子来计算自己的新种子，依此类推，只有保证了这个，才能保证在多线程下产生的随机数是随机的。

```java
public int nextInt() {
	return next(32);
}

protected int next(int bits) {
	long oldseed, nextseed;
	AtomicLong seed = this.seed;
	do {
	oldseed = seed.get();
		nextseed = (oldseed * multiplier + addend) & mask;
	} while (!seed.compareAndSet(oldseed, nextseed));
	return (int)(nextseed >>> (48 - bits));
}
```

next函数中的seed是一个原子Long，采用cas操作保证了只有一个线程可以更新新的seed。其余线程全部自旋寻找新的seed。

这样会消耗很多性能，所以才有了ThreadLocalRandom类。

## 3.2 ThreadlocalRandom

random性能差是因为维护了一个原子seed，ThreadLocalRandom和ThreadLocal类似，为每个Thread维护一个seed，这样将大大提高并发量。

当调用ThreadLocalRandom的nextlnt方法时，实际上是获取当前线程的threadLocalRandomSeed变量作为当前种子来计算新的种子，然后更新新的种子到当前线程的threadLoca!RandomSeed变量，而后再根据新种子并使用具体算法计算随机数。这里需要注意的是，threadLoca!RandomSeed变量就是Thread类里面的一个普通long变量，它并不是原子性变量。其实道理很简单，因为这个变量是线程级别的，所以根本不需要使用原子性变量。

ThreadLocalRandom维护一个static的isstance，下面来看一下如何实现高性能的并发。

```java
 /** The common ThreadLocalRandom */
    static final ThreadLocalRandom instance = new ThreadLocalRandom();
    public static ThreadLocalRandom current() {
        if (U.getInt(Thread.currentThread(), PROBE) == 0)
            localInit();
        return instance;
    }

    static final void localInit() {
        int p = probeGenerator.addAndGet(PROBE_INCREMENT);
        int probe = (p == 0) ? 1 : p; // skip 0
        long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
        Thread t = Thread.currentThread();
        U.putLong(t, SEED, seed);
        U.putInt(t, PROBE, probe);
    }
```

这里使用到了“延迟加载”技术来优化，我们主要关注localInit方法里面的最后三句，将数据绑定到了各个thread里面的SEED（偏移量）和PROBE（偏移量）。*这里为什么要用Unsafe类呢，也就是为什么要用原子操作？*

现在调用nextInt（）：

```java
    public int nextInt(int bound) {
        if (bound <= 0)
            throw new IllegalArgumentException(BAD_BOUND);
        int r = mix32(nextSeed());
        int m = bound - 1;
        if ((bound & m) == 0) // power of two
            r &= m;
        else { // reject over-represented candidates
            for (int u = r >>> 1;
                 u + m - (r = u % bound) < 0;
                 u = mix32(nextSeed()) >>> 1)
                ;
        }
        return r;
    }
    final long nextSeed() {
        Thread t; long r; // read and update per-thread seed
        U.putLong(t = Thread.currentThread(), SEED,
                  r = U.getLong(t, SEED) + GAMMA);
        return r;
    }

```

ThreadLocalRandom总结：在多线程下计算新种子时是根据自己线程内维护的种子变量进行更新，从而避免
了竞争。



