DelayQueue即延迟队列。也就是一个按延迟时间从小到大出队的PriorityQueue。所谓延迟时间，就是“未来将要执行的时间 - 当前时间”。为此，放入DelayQueue中的元素，必须实现Delayed接口。如下所示

```java
public interface Delayed extends Comparable<Delayed> {
    /*
     * 1、如果getDelay()的返回值小于或等于0，说明该元素到期，需要从队列中拿出来执行
     * 2、接口继承了Comparable，所以要实现该接口，必须先实现Comparable接口。具体来说就是基于getDelay()返回值比较两个元素的大小
     */
    long getDelay(TimeUnit unit);
}
```

DelayQueue的核心数据结构：

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E> implements BlockingQueue<E> {
    // 一个优先级队列
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    //“一个锁 + 一个条件”控制是否操作阻塞
    private final transient ReentrantLock lock = new ReentrantLock();
    private final Condition available = lock.newCondition();
}
```

take操作

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek(); // 取出二叉堆的堆顶元素，也就是延迟时间最小的
            if (first == null){
                available.await();//队列为空，take线程阻塞
            } else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0){ // 堆顶元素的延迟时间小于或等于0，出队列，返回
                    return q.poll();
                }
                first = null; 
                if (leader != null){    //如果已经有其他线程也在等待这个元素，则无限期阻塞
                    available.await();
                } else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay); //否则阻塞有限的时间
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null){ // 自己是leader，已经获取了堆顶元素，唤醒其他线程
            available.signal();
        }
        lock.unlock();
    }
}
```

关于take方法的说明

- 不同于一般的阻塞队列，在队列为空或者堆顶元素的延迟时间没到，才会阻塞。
- 上面的代码中，使用一个Thread leader变量记录了等待堆顶元素的第一个线程。为什么这么做呢？通过getDelay(...)可以知道堆顶元素何时到期，不必无限期等待，可以使用condition.awaitNanos()等待一个有限时间；只有当发现还有其他线程也在等待堆顶元素(leader !=NULL)时，才无限期等待

put操作

```java
public void put(E e) {
    offer(e);
}

public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        q.offer(e);			//元素放入二叉堆
        if (q.peek() == e) {//如果放进去的元素刚好在堆顶，说明放入的元素延迟时间最小，那么通知等待的线程，否则放入的元素不在堆顶位置就不通知
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```

put的说明：

- 不是每放入一个元素，都需要通知等待的线程。放入的元素，如果其延迟时间大于当前堆顶元素延迟时间，就没有必要通知等待的线程
- 只有延迟时间是最小的，在堆顶时，才有必要通知等待的线程。即上面if (q.peek() == e) {}的代码片段。



