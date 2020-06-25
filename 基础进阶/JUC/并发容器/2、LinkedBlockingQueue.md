LinkedBlockingQueue是一个基于单向链表的阻塞队列。

由于队头和队尾分别使用2个指针分开操作，因此用了2把锁和2个条件，同时使用AtomicInteger的原子变量记录count个数。

LinkedBlockingQueue的核心数据结构：

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable {
    private final AtomicInteger count = new AtomicInteger();//当前元素个数
    transient Node<E> head;
    private transient Node<E> last;

    private final ReentrantLock takeLock = new ReentrantLock();// take、poll等操作使用的锁
    private final Condition notEmpty = takeLock.newCondition(); //take、poll等操作使用条件
    private final ReentrantLock putLock = new ReentrantLock();//put、offer等操作使用的锁
    private final Condition notFull = putLock.newCondition();//put、offer等操作的使用条件
    
    //链表节点定义
    static class Node<E> {
    	E item;
    	Node<E> next;
    	Node(E x) { item = x; }
	}
}
```

构造函数也可以支持指定容量，如果不指定，默认为Integer.MAX_VALUE

```java
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);
}
```

核心的take和put操作的具体实现

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock; 
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();			// 使用到put操作的可中断锁
    try {
        while (count.get() == capacity) {
            notFull.await();				//队列满了，阻塞所有put操作的线程
        }
        enqueue(node);						//put将元素放入队列
        c = count.getAndIncrement();		//元素个数加1
        if (c + 1 < capacity) 
            notFull.signal();				//同时其他put操作
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}


public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            notEmpty.await();		//队列空了，非空条件阻塞所有take操作的线程
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

LinkedBlockingQueue有几点说明：

- 为了提高并发度，用了2把锁，分别控制队头、队尾的操作。意味着在put()和put()之间、take()和take()操作之前是互斥的，put()和take()操作不互斥。但对于count变量，双方都需要操作，所以必须是原子类型。

- 由于各自拿了一把锁，当需要调用对方的condition的signal时，还必须再加上对方的锁，也就是signalNotEmpty和signalNotFull方法。

    ```java
    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;	// 必须先拿到putLock，才能调用notFull.signal
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }
    
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;//必须先拿到takeLock，才能调用notEmpty.signal
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
    ```

- 不仅put会通知take，take也会通知put。当put发现非满的时候，也会通知其他put线程；当take发现非空的时候，也会通知其他take线程。



