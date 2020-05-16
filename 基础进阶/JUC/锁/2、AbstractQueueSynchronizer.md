## 可重入锁

可重复锁就是获取锁的线程可以重复对资源进行加锁，同时也支持获取锁时公平和非公平的选择。

#### 1、公平锁与非公平锁

如果绝对时间上来说，先对锁进行获取的请求一定会先被满足，那么这个锁是公平的，反之是非公平的。

ReentrantLock默认是非公平锁，除非在构造函数传入true来构建公平锁。

```java
public ReentrantLock() {
    sync = new NonfairSync();//默认非公平锁
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

#### 2、实现可重入关键点

- 线程再次获取到锁：锁需要去识别获取锁的线程是否是当前占据锁的线程，如果是，则再次获取成功。
- 锁的最终释放：线程N次获取到锁，释放的时候需要N次释放，这样其他线程能够获取到锁。意味着，锁需要通过计数器进行控制。

ReentrantLock中获取公平锁：

```java
static final class FairSync extends Sync {
    final void lock() {
        acquire(1);
    }
    //AbstractQueuedSynchronizer.acquire()
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 1. 和非公平锁相比，这里多了一个判断：是否有线程在等待
            if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

ReentrantLock中获取非公平锁

```java
static final class NonfairSync extends Sync {
    final void lock() {
        //和公平锁相比，这里会直接先进行一次CAS，成功就返回了
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
    
    /**
     * 继承自AbstractQueuedSynchronizer的acquire()
     */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)){
            selfInterrupt();
        }
    }
    /**
     * 尝试获取锁
     */
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

/**
 * 非功能锁的获取
 */
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 这里没有对阻塞队列进行判断
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

#### 3、总结

公平锁和非公平锁只有两处不同：

- 非公平锁在调用 lock 后，首先就会调用 CAS 进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。
- 非公平锁在 CAS 失败后，和公平锁一样都会进入到 tryAcquire 方法，在 tryAcquire 方法中，如果发现锁这个时候被释放了（state= 0），非公平锁会直接 CAS 抢锁，但是公平锁会判断等待队列是否有线程处于等待状态，如果有则不去抢锁，乖乖排到后面。

公平锁和非公平锁就这两点区别，如果这两次 CAS 都不成功，那么后面非公平锁和公平锁是一样的，都要进入到阻塞队列等待唤醒。相对来说，非公平锁会有更好的性能，因为它的吞吐量比较大。当然，非公平锁让获取锁的时间变得更加不确定，可能会导致在阻塞队列中的线程长期处于饥饿状态。

