[TOC]

CopyOnWriteArrayList是一个线程安全的ArrayList，对其进行的修改操作都是在底层的一个复制的数组（快照）上进行的，也就是使用了写时复制策略。

![](https://pic.downk.cc/item/5e63743c98271cb2b8d7c714.jpg)

每个CopyOnWriteArrayList对象里面有一个array数组对象用来存放具体元素，ReentrantLock独占锁对象用来保证同时只有－个线程对aηay进行修改。

如果让我们自己做一个写时复制的线程安全的list 我们会怎么做，有哪些点需要考虑？

- 何时初始化list ，初始化的list 元素个数为多少， list 是有限大小吗？
- 如何保证线程安全，比如多个线程进行读写时如何保证是线程安全的？
- 如何保证使用法代器遍历list 时的数据一致性？

## 5.2 主要方法源码解析

### 5.2.1 初始化

```java

 public CopyOnWriteArrayList() {
 	 setArray(new Object[0]);
}

public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] es;
    if (c.getClass() == CopyOnWriteArrayList.class)
        es = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        es = c.toArray();
        // defend against c.toArray (incorrectly) not returning Object[]
        // (see e.g. https://bugs.openjdk.java.net/browse/JDK-6260652)
        if (es.getClass() != Object[].class)
            es = Arrays.copyOf(es, es.length, Object[].class);
    }
    setArray(es);
}

public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}

final void setArray(Object[] a) {
	array = a;
}
```
### 5.2.2 添加元素

```java

    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        synchronized (lock) {
            Object[] es = getArray();
            int len = es.length;
            es = Arrays.copyOf(es, len + 1);
            es[len] = e;
            setArray(es);
            return true;
        }
    }
```

只用注意所有的操作都是COW操作就行。

### 5.2.3 获取指定位置元素

get操作：

```java
    /**
     * {@inheritDoc}
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        return elementAt(getArray(), index);
    }
    final Object[] getArray() {
        return array;
    }
    @SuppressWarnings("unchecked")
    static <E> E elementAt(Object[] a, int index) {
        return (E) a[index];
    }
```

考虑这里为什么没有加锁。如果有另一个线程对array做了write操作呢？这里其实是没必要的，是一个弱引用的问题。

![](https://pic.downk.cc/item/5e6378ad98271cb2b8d9cd01.jpg)

由于执行步骤A和步骤B没有加锁，这就可能导致在线程x执行完步骤A后执行步骤B前，另外一个线程y进行了remove操作，假设要删除元素1oremove操作首先会获取独占锁，然后进行写时复制操作，也就是复制一份当前array数组，然后在复制的数组里面删除线程x通过get方法要访问的元素1，之后让array指向复制的数组。而这时候aηay之前指向的数组的引用计数为l而不是0，因为线程x还在使用它，这时线程x开始执行步骤B，步骤B操作的数组是线程y删除元素之前的数组.

### 5.2.4 修改指定元素

```java

    /**
     * Replaces the element at the specified position in this list with the
     * specified element.
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E set(int index, E element) {
        synchronized (lock) {
            Object[] es = getArray();
            E oldValue = elementAt(es, index);

            if (oldValue != element) {
                es = es.clone();
                es[index] = element;
            }
            // Ensure volatile write semantics even when oldvalue == element
            setArray(es);
            return oldValue;
        }
    }

```

没什么好说的。

### 5.2.5 删除元素

也是先获得锁，然后进行操作。

### 5.2.6 弱一致性的迭代器

说明获取迭代器后， 使用该法代器元素时， 其他线程对该list 进行的增删改不可见，因为它们操作的是两个不同的数组， 这就是弱一致。

**为什么呢？**因为其它线程会用一个副本来修改这个list，而iteraotr中的引用指向了老的array。所以iterator不受印象。

```java

    static final class COWIterator<E> implements ListIterator<E> {
        /** Snapshot of the array */
        private final Object[] snapshot;	
        /** Index of element to be returned by subsequent call to next.  */
        private int cursor;

        COWIterator(Object[] es, int initialCursor) {
            cursor = initialCursor;
            snapshot = es;		// 注意这里其实是传递的引用，但是由于cow机制，所以其余线程并不影响iterator
        }

        public boolean hasNext() {
            return cursor < snapshot.length;
        }

        public boolean hasPrevious() {
            return cursor > 0;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }
    }

```

