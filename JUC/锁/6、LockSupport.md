LockSupport定义了一组公共的静态方法，这些方法提供了基本的线程阻塞和唤醒功能，而LockSupport也成为构建同步组件的基础工具。

#### 1、LockSupport提供的阻塞和唤醒方法

| **方法名称**                  | **方法描述**                                                 |
| :---------------------------- | :----------------------------------------------------------- |
| void park()                   | 阻塞当前线程。如果调用unpark(Thread thread)方法或线程被中断，才能从park方法返回 |
| void park(long nanos)         | 阻塞当前线程，最长不超过nanos纳秒，返回条件在park基础上加了超时返回 |
| void parkUntil(long deadline) | 阻塞当前线程，直到deadline时间（从1970年到deadline的时间毫秒数） |
| void unpark(Thread thread)    | 唤醒阻塞状态的线程                                           |

#### 2、LockSupport原理

最多允许一个许可证令牌（permit），如果线程可以得到这个令牌，那么它就能继续运行下去，否则就阻塞状态。此时线程状态时waiting而不是blocked

#### 3、park系列方法

以park为前缀的方法都是用来获取许可证的，如果当前线程的许可证令牌存在一个，那么调用park系列方法的线程会立即返回继续执行，同时许可证的可用个数就变成0，如果再继续调用park方法，那么当前线程就会阻塞处于等待状态，除非有以下状态就会才会唤醒线程：

- 任何其他线程对该线程对象调用了unpark归还了令牌，就会接触阻塞。许可证令牌只有一个，不会随着unpark方法调用而随之增加。
- 任何其他线程对该线程调用了interrupt方法，打断了线程
- 由于操作系统不可预知的原因，虚假唤醒
- 调用了以下方法，在过期时间到达之后自动唤醒
  - parkNanos(long nanos)
  - parkNanos(Object blocker, long nanos)
  - parkUntil(long deadline)
  - parkUntil(Object blocker, long deadline)

#### 4、unpark方法

调用这个方法可以保证归还一个令牌，从而保证下一次调用park方法的线程可以直接立即返回运行。



#### 5、park和getBlocker方法对象参数的作用

park方法的blocker参数作用时传入一个对象，记录监控信息，这样当线程阻塞的时候，可以调用getBlocker方法读取这个对象，从而了解一些阻塞线程的信息，这里获取的是最后一次park的对象信息，因为park可以调用无数次，所以这里只能保证获取最新的自定义的对象监控信息。



#### 6、相关问题

- 调用park方法会使得当前线程出于WAITING或WAITING_TIMED状态，当调用unpark并不会立即进入RUNNABLE状态，而是看当前有没有现成已经获取到锁，如果有，它们会变成BLOCKED状态然后变成RUNNABLE状态。
- WAITING和BLOCKED状态的区别：WAITING一般需要通知才能唤醒；而BLOCKED通常是只要对象的锁没有被占用就可以继续执行，如果有线程占用了锁对象，就进入BLOCKED。
- LockSupport一般不会直接出现在代码里，绝大多数时候只需要使用AbstractQueuedSynchronizer这个封装好的工具类即可。
- park和unpark在官方说明书中没有要求他们happened-before规则，因此他们不能保证相关的原子性、可见性和有序性。



