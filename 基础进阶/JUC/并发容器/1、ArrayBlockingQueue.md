ArrayBlockingQueue是一个用数组实现的环形队列，在构造函数中，会要求传入数组的容量。

```java
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

ArrayBlockingQueue的核心数据结构是

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable {
    final Object[] items;				//存放元素的数组
    int takeIndex;						//下一次take操作的索引位置
    int putIndex;						//下一次put操作的索引位置
    int count;							//队列中实际存储的元素个数
    final ReentrantLock lock;			//保证所有操作的一把lock锁
    private final Condition notEmpty; 	//take操作的Condition条件
    private final Condition notFull;	//put操作的Condition条件
}
```

可以看到ArrayBlockingQueue的核心是`一把锁+两个条件`。put/take操作也很简单

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();		// 使用可中断的锁
    try {
        while (count == items.length)
            notFull.await();		//若队列满了，则操作阻塞
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

private void enqueue(E x) {
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    notEmpty.signal();			//put进去之后，通知“非空”条件
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();	//若队列为空，则操作阻塞
        return dequeue();
    } finally {
        lock.unlock();
    }
}
private E dequeue() {
    final Object[] items = this.items;
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();			//take结束，通知“非满”条件
    return x;
}
```

