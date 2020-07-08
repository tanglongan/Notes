队列是先进先出的，而PriorityBlockingQueue是按照元素的优先级从小到大出队列的。

正因如此，PriorityBlockingQueue中2个元素之间需要可以比较大小，并实现Comparable接口。

核心数据结构：

```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable {
    private transient Object[] queue;
    private transient int size;
    private transient Comparator<? super E> comparator;
    // 一把锁 + 一个条件
    private final ReentrantLock lock;
    private final Condition notEmpty;
}
```

构造函数如下，如果不指定大小，内部会设置一个默认值11，当元素个数超过这个大小之后，会自动扩容

```java
private static final int DEFAULT_INITIAL_CAPACITY = 11;
public PriorityBlockingQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}
public PriorityBlockingQueue(int initialCapacity,Comparator<? super E> comparator) {
    if (initialCapacity < 1) throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    this.queue = new Object[initialCapacity];
}
```

take和put操作

```java
public void put(E e) {
	offer(e);
}

public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);		// 当size超过了数组长度，扩容
    try {
        Comparator<? super E> cmp = comparator;
        if (cmp == null){					// 没有定义比较操作符，使用功能元素自带的比较功能
            siftUpComparable(n, e, array);	//元素入堆，也就是执行siftUp操作
        } else{
            siftUpUsingComparator(n, e, array, cmp);
        }
        size = n + 1;
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}


public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null) // 出队列
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}

private E dequeue() {
    int n = size - 1;
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        E result = (E) array[0];  //因为是最小二叉堆，堆定就是最小的元素
        E x = (E) array[n];
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftDownComparable(0, x, array, n);			//元素出堆之后，调整堆，siftDown操作
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}

```

从上面的源码可见，和ArrayBlockingQueue的机制相似，主要区别是用数组实现了一个二叉堆，从而实现了按优先级从小到大出队列。另一个区别是没有notFull条件，当元素个数超过数组条件时，执行扩容操作。