[TOC]

JDK中提供了一系列场景的并发安全队列。总的来说，按照实现方式的不同可分为阻塞队列和非阻塞队列，前者使用锁实现，而后者则使用CAS非阻塞算法实现。

## 7.1 ConcurrentlinkedQueue 原理探究

ConcurrentLinkedQueue是线程安全的**无界非阻塞队列**，其底层数据结构使用单向链表实现，对于入队和出队操作使用**CAS来实现线程安全**。

先看类图：

![](https://pic.downk.cc/item/5e6597aa98271cb2b8de9501.jpg)

ConcurrentLinkedQueue是线程安全的无界非阻塞队列，其底层数据结构使用单向链表实现，对于入队和出队操作使用CAS来实现线程安全。在Node节点内部则维护一个使用volatile修饰的变量item，用来存放节点的值；next用来存放链表的下一个节点，从而链接为一个单向无界链表。其内部则使用UNSafe工具类提供的CAS算法来保证出入队时操作链表的原子性。

看看ConcurrentLinkedQueue的构造函数：

```java
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>();
}
```
**head和tail是一个哨兵节点。**

### 7.1.2 ConcurrentlinkedQueue 原理介绍

#### 1.offer操作

```java

   public boolean offer(E e) {
        final Node<E> newNode = new Node<E>(Objects.requireNonNull(e));

        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            if (q == null) {	// 代码1
                // p is last node
                if (NEXT.compareAndSet(p, null, newNode)) {
                    // Successful CAS is the linearization point
                    // for e to become an element of this queue,
                    // and for newNode to become "live".
                    if (p != t) // hop two nodes at a time; failure is OK // 代码2
                        TAIL.weakCompareAndSet(this, t, newNode);
                    return true;
                }
                // Lost CAS race to another thread; re-read next
            }
            else if (p == q)		// 代码 3
                // We have fallen off list.  If tail is unchanged, it
                // will also be off-list, in which case we need to
                // jump to head, from which all live nodes are always
                // reachable.  Else the new tail is a better bet.
                p = (t != (t = tail)) ? t : head;
            else			// 代码4
                // Check for tail updates after two hops.
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }

```



**代码1,2,4 比较好理解，就是判断是否是尾结点，插入，重置tail，或者循环找尾节点。但是注意代码1处如果有两个线程同时调用了cas，只有只有其中一个线程可以正确设置，更新了结构状态，另一线程的cas返回false，于是继续循环找到更新后的结构尾节点。再进行cas。**

难以理解的是代码3：

书上说，在执行poll操作后才会执行到代码3。这里先来看一下执行poll操作后可能会存在的一种情况：

![](https://pic.downk.cc/item/5e659d5498271cb2b8e2f853.jpg)

![](https://pic.downk.cc/item/5e659d7798271cb2b8e30e22.jpg)

要在执行poll操作后才会执行。这里先来看一下执行poll操作后可能会存在的一种情况，如图7-8所示这里由于q节点不为空并且p==q所以执行代码（7），由于t==tail所以p被赋值为head，然后重新循环，循环后执行到代码（的，这时候队列状态如图7-10所示。

![](https://pic.downk.cc/item/5e659d9d98271cb2b8e31858.jpg)

这时候由于q==null，所以执行代码（引进行CAS操作，如果当前没有其他线程执行offer操作，则CAS操作会成功，p的next节点被设置为新增节点。然后执行代码（6)'由于p!=t所以设置新节点为队列的尾部节点，现在队列状态如图7-11所示。

![](https://pic.downk.cc/item/5e659db898271cb2b8e32281.jpg)

需要注意的是，这里自引用的节点会被垃圾回收掉。

#### 2.add 操f乍

add 操作是在链表末尾添加一个元素，其实在内部调用的还是offer 操作。

```java

    public boolean add(E e) {
        return offer(e);
    }

```

#### 3.poll 操f乍

![](https://pic.downk.cc/item/5e65a00298271cb2b8e43d44.jpg)

```java

    public E poll() {
        restartFromHead: for (;;) {
            for (Node<E> h = head, p = h, q;; p = q) {
                final E item;
                if ((item = p.item) != null && p.casItem(item, null)) {
                    // Successful CAS is the linearization point
                    // for item to be removed from this queue.
                    if (p != h) // hop two nodes at a time
                        updateHead(h, ((q = p.next) != null) ? q : p);
                    return item;
                }
                else if ((q = p.next) == null) {
                    updateHead(h, p);
                    return null;
                }
                else if (p == q)		// 这个需要两个线程同时操作才有可能发生
                    continue restartFromHead;
            }
        }
    }

```

**总结：**poll方法在移除一个元素时，只是简单地使用CAS操作把当前节点的item值设置为null，然后通过重新设置头节点将该元素从队列里面移除，被移除的节点就成了孤立节点，这个节点会在垃圾回收时被回收掉。另外，如果在执行分支中发现头节点被修改了，要跳到外层循环重新获取新的头节点。

#### 4.peek操作

```java

    public E peek() {
        restartFromHead: for (;;) {
            for (Node<E> h = head, p = h, q;; p = q) {
                final E item;
                if ((item = p.item) != null
                    || (q = p.next) == null) {
                    updateHead(h, p);	
                    return item;
                }
                else if (p == q)
                    continue restartFromHead;
            }
        }
    }

    /**
     * Tries to CAS head to p. If successful, repoint old head to itself
     * as sentinel for succ(), below.
     */
    final void updateHead(Node<E> h, Node<E> p) {
        // assert h != null && p != null && (h == p || h.item == null);
        if (h != p && HEAD.compareAndSet(this, h, p))
            NEXT.setRelease(h, h);	// 自引用
    }

```

总结：peek操作的代码与poll操作类似，只是前者只获取队列头元素但是并不从队列里将它删除，而后者获取后需要从队列里面将它删除。另外，**在第一次调用peek操作时，会删除哨兵节点，并让队列的head节点指向队列里面第一个元素或者null。**

#### 6.remove 操作

如果队列里面存在该元素则删除该元素，如果存在多个则删除第一个，并返回true ,否则返回false 。

```java

    /**
     * Removes a single instance of the specified element from this queue,
     * if it is present.  More formally, removes an element {@code e} such
     * that {@code o.equals(e)}, if this queue contains one or more such
     * elements.
     * Returns {@code true} if this queue contained the specified element
     * (or equivalently, if this queue changed as a result of the call).
     *
     * @param o element to be removed from this queue, if present
     * @return {@code true} if this queue changed as a result of the call
     */
    public boolean remove(Object o) {
        if (o == null) return false;
        restartFromHead: for (;;) {
            for (Node<E> p = head, pred = null; p != null; ) {
                Node<E> q = p.next;
                final E item;
                if ((item = p.item) != null) {
                    if (o.equals(item) && p.casItem(item, null)) {
                        skipDeadNodes(pred, p, p, q);
                        return true;
                    }
                    pred = p; p = q; continue;
                }
                for (Node<E> c = p;; q = p.next) {
                    if (q == null || q.item != null) {
                        pred = skipDeadNodes(pred, c, p, q); p = q; break;
                    }
                    if (p == (p = q)) continue restartFromHead;
                }
            }
            return false;
        }
    }

    /**
     * Collapse dead nodes between pred and q.
     * @param pred the last known live node, or null if none
     * @param c the first dead node
     * @param p the last dead node
     * @param q p.next: the next live node, or null if at end
     * @return either old pred or p if pred dead or CAS failed
     */
    private Node<E> skipDeadNodes(Node<E> pred, Node<E> c, Node<E> p, Node<E> q) {
        // assert pred != c;
        // assert p != q;
        // assert c.item == null;
        // assert p.item == null;
        if (q == null) {
            // Never unlink trailing node.
            if (c == p) return pred;
            q = p;
        }
        return (tryCasSuccessor(pred, c, q)
                && (pred == null || ITEM.get(pred) != null))
            ? pred : p;
    }


```

### 7.1.3 小结

ConcurrentLinkedQueue的底层使用**单向链表数据结构来保存队列元素**，每个元素被包装成一个Node节点。队列是靠头、尾节点来维护的，创建队列时头、尾节点指向个item为null的哨兵节点。第一次执行peek或者自remove操作时会把head指向第一个真正的队列元素。由于**使用非阻塞CAS算法，没有加锁**，所以在计算size时有可能进行了offer、poll或者remove操作，导致计算的元素个数不精确，所以在井发情况下size函数不是很有用。

如图7-27所示，**入队、出队都是操作使用volatile修饰的tail、head节点，**要保证在多线程下出入队线程安全，只需要保证这两个Node操作的可见性和原子性即可。**由于volatile本身可以保证可见性，所以只需要保证对两个变量操作的原子性即可。**

![](https://pic.downk.cc/item/5e65a2e098271cb2b8e59ff4.jpg)

offer操作是在tail后面添加元素，也就是调用tail.casNext方法，而这个方法使用的是CAS操作，只有一个线程会成功，然后失败的线程会循环，重新获取tail，再执行casNext方法。poll操作也通过类似CAS的算法保证出队时移除节点操作的原子性。

## 7 .2 LinkedBlockingQueue 原理探究

先看类图：

![](https://pic.downk.cc/item/5e65a39d98271cb2b8e5f464.jpg)

有两个ReentrantLock的实例，分别用来控制元素入队和出队的原子性，其中takeLock用来控制同时只有一个线程可以从队列头获取元素，其他线程必须等待，putLock控制同时只能有一个线程可以获取锁，在队列尾部添加元素，其他线程必须等待。另外，notEmpty和notFull是条件变量，它们内部都有一个条件队列用来存放进队和出队时被阻塞的线程，其实这是生产者一消费者模型。如下是独占锁的创建代码。

### 7.2.2 LinkedBlockingQueue 原理介绍

#### 1.offer操作

**该方法不阻塞**

```java

    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        if (count.get() == capacity)
            return false;
        final int c;
        final Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            if (count.get() == capacity)
                return false;
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();	// 唤醒因为满而阻塞的线程。
        } finally {
            putLock.unlock();
        }
        if (c == 0)		// 为什么要用c == 0 ？ 因为顺利执行了count.getAndIncrement()，所以queue现在有element了？
            signalNotEmpty();	// 唤醒因为queue空而阻塞的线程。
        return true;
    }

```
offer方法通过使用putLock锁保证了在队尾新增元素操作的原子’性。另外，调用条件变量的方法前一定要记得获取对应的锁，并且注意进队时只操作队列链表的尾节点。

#### 2. put 操作

put操作和offer操作不同的在于，put时如果发现capacity已经满了，那么将会阻塞当前线程，直到插入成功。另外就是put操作可被中断，offer不可中断。**该方法阻塞。**

```

    /**
     * Inserts the specified element at the tail of this queue, waiting if
     * necessary for space to become available.
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        final int c;
        final Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }

```

#### 3. poll 操作

方法不阻塞。

这里不贴源码了。

#### 4.peek 操作

#### 5. take 操作

该方法会阻塞

#### 6.remove操作

该方法会获取takelock和putlock，由于remove方法在删除指定元素前加了两把锁，所以在遍历队列查找指定元素的过程中是线程安全的，并且此时其他调用入队、出队操作的线程全部会被阻塞。另外，获取多个资源锁的顺序与释放的顺序是相反的。

#### 7.size 操作

```java
public int size(){
    return count.get();
}
```

由于进行出队、入队操作时的count是加了锁的，所以结果相比ConcurrentLinkedQueue的size方法**比较准确**。这里考虑为何在ConcurrentLinkedQueue中需要遍历链表来获取size而不使用一个原子变量呢？这是因为使用原子变量保存队列元素个数需要保证入队、出队操作和原子变量操作是原子性操作，而ConcurrentLinkedQueue使用的是CAS无锁算法，所以无法做到这样。

### 总结：

![](https://pic.downk.cc/item/5e65b6f598271cb2b8ee657c.jpg)

LinkedBlockingQueue的内部是**通过单向链表实现的**，使用头、尾节点来进行入队和出队操作，也就是入队操作都是对尾节点进行操作，出队操作都是对头节点进行操作。如图7-29所示，**对头、尾节点的操作分别使用了单独的独占锁从而保证了原子性**，所以出队和入队操作是可以同时进行的。另外对头、尾节点的独占锁都配备了一个条件队列，用来存放被阻塞的线程，并结合入队、出队操作**实现了一个生产消费模型**。

## 7.3 ArrayBlockingQueue 原理探究

ArrayBlockingQueue采用数组方式来实现queue。

### 7.3.1 类图结构

![](https://pic.downk.cc/item/5e65b7c498271cb2b8eea618.jpg)

ArrayBlockingQueue在默认情况下使用ReentrantLock 提供的非公平独占锁进行出、入队操作的同步。

#### 1 . offer 操作

该方法是不阻塞的。

```java

    public boolean offer(E e) {
        Objects.requireNonNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length)
                return false;
            else {
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
```

代码还是相当简单的，获得锁，判定是否满，不满则入，满则返回false。

#### 2. put 操作

和LinkedBlockingQueue的put同理，put操作是阻塞的，只要队列是满的，就一直阻塞，找到可入队。

#### 3. poll 操作

对应offer，不阻塞，队列空则null。

```java

   public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
    /**
     * Extracts element at current take position, advances, and signals.
     * Call only when holding lock.
     */
    private E dequeue() {
        // assert lock.isHeldByCurrentThread();
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E e = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length) takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return e;
    }
```

#### 4. take操作

和put对应，只不过是阻塞的。

#### 5. peek 操作



### 总结：

ArrayBlockingQueue通过使用全局独占锁实现了同时只能有一个线程进行入队或者出队操作，这个锁的粒度比较大，有点类似于在方法上添加synchronized的意思。其中。他r和poll操作通过简单的加锁进行入队、出队操作，而put、take操作则使用条件变量实现了，如果队列满则等待，如果队列空则等待，然后分别在出队和入队操作中发送信号激活等待线程实现同步。另外，**相比LinkedBlockingQueue,ArrayBlockingQueue的size操作的结果是精确的**，因为计算前加了全局锁。

![](https://pic.downk.cc/item/5e65c42898271cb2b8f2d735.jpg)

## 7.4 PriorityBlockingQueue 原理探究

PriorityBlockingQueue是带优先级的无界阻塞队列，每次出队都返回优先级最高或者最低的元素。其内部是使用平衡二叉树堆实现的，所以直接遍历队列元素不保证有序。默认使用对象的compareTo方法提供比较规则，如果你需要自定义比较规则则可以自定义comparators。

![](https://pic.downk.cc/item/5e65c4a798271cb2b8f2f698.jpg)

### 7.4.3 原理介绍

#### 1.offer 操作

```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] es;
    while ((n = size) >= (cap = (es = queue).length))
        tryGrow(es, cap);		// 在这里扩容
    try {
        final Comparator<? super E> cmp;
        if ((cmp = comparator) == null)
            siftUpComparable(n, e, es);
        else
            siftUpUsingComparator(n, e, es, cmp);
        size = n + 1;
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}


    /**
     * Tries to grow array to accommodate at least one more element
     * (but normally expand by about 50%), giving up (allowing retry)
     * on contention (which we expect to be rare). Call only while
     * holding lock.
     *
     * @param array the heap array
     * @param oldCap the length of the array
     */
    private void tryGrow(Object[] array, int oldCap) {
        lock.unlock(); // must release and then re-acquire main lock
        // 这里释放锁所用cas的原因是因为扩容操作耗时较长，采用cas可以让其他线程继续入队出队，提高并发量。
        Object[] newArray = null;
        if (allocationSpinLock == 0 &&
            ALLOCATIONSPINLOCK.compareAndSet(this, 0, 1)) {
            try {
                int newCap = oldCap + ((oldCap < 64) ?
                                       (oldCap + 2) : // grow faster if small
                                       (oldCap >> 1));
                if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                    int minCap = oldCap + 1;
                    if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                        throw new OutOfMemoryError();
                    newCap = MAX_ARRAY_SIZE;
                }
                if (newCap > oldCap && queue == array)
                    newArray = new Object[newCap];
            } finally {
                allocationSpinLock = 0;	// 这里没有采用cas，是因为只有一一个线程可以进行扩容
            }
        }
        if (newArray == null) // back off if another thread is allocating
            Thread.yield();
        lock.lock();
        if (newArray != null && queue == array) {
            queue = newArray;	 
            System.arraycopy(array, 0, newArray, 0, oldCap);
        }
    }

```
再来看看建堆过程，heap：

```java
    private static <T> void siftUpComparable(int k, T x, Object[] es) {
        Comparable<? super T> key = (Comparable<? super T>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = es[parent];
            if (key.compareTo((T) e) >= 0)
                break;
            es[k] = e;
            k = parent;
        }
        es[k] = key;
    }
```

#### 2.poll操作

出队第一元素，然后更新heap，shiftDown。可能为null。

#### 3. put 操作

同offer

#### 4.take操作

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null)	// 记住所有的条件变量都应该使用while来判定，避免虚假唤醒
            notEmpty.await();		// 空则等待offer或put操作
    } finally {
        lock.unlock();
    }
    return result;
}
```
### 总结：

PriorityBlockingQueue队列在内部使用二叉树堆维护元素优先级，使用数组作为元素存储的数据结构，这个数组是可扩容的。当当前元素个数＞＝最大容量时会通过CAS算法扩容，出队时始终保证出队的元素是堆树的根节点，而不是在队列里面停留时间最长的元素。使用元素的compareTo方法提供默认的元素优先级比较规则，用户可以自定义优先级的比较规则。

![](https://pic.downk.cc/item/5e65cd0b98271cb2b8fc0396.jpg)

## 7.5 DelayQueue 原理探究

DelayQueue并发队列是一个无界阻塞延迟队列，队列中的每个元素都有个过期时间，当从队列获取元素时，只有过期元素才会出队列。队列头元素是最快要过期的元素。

![](https://pic.downk.cc/item/5e65d0a598271cb2b80009b7.jpg)

### 7.5.2 主要函数原理讲解

其中leader 变量的使用**基于Leader-Follower 模式**的变体。估计是个设计模式。

#### 1.offer操作

```java
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {		// 检查当前插入的e是不是最先过期的
                leader = null;
                available.signal();		// 唤醒一个follower
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
```

#### 2.take操作

![](https://pic.downk.cc/item/5e65d23298271cb2b800b2f4.jpg)

![](https://pic.downk.cc/item/5e65d24798271cb2b800b907.jpg)

![](https://pic.downk.cc/item/5e65d25b98271cb2b800bf14.jpg)

![](https://pic.downk.cc/item/5e65d26a98271cb2b800c53c.jpg)

#### 3.poll 操f乍

获取并移除队头过期元素，如果没有过期元素则返回null 。

```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            E first = q.peek();
            return (first == null || first.getDelay(NANOSECONDS) > 0)
                ? null
                : q.poll();
        } finally {
            lock.unlock();
        }
    }
```

###  总结:

其内部使用PriorityQueue存放数据，使用ReentrantLock实现线程同步。另外队列里面的元素要实现Delayed接口，其中一个是获取当前元素到过期时间剩余时间的接口，在出队时判断元素是否过期了，一个是元素之间比较的接口，因为这是一个有优先级的队列。

![](https://pic.downk.cc/item/5e65d34798271cb2b80109a2.jpg)



### 