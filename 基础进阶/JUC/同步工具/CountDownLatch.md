CountDownLatch可以看做是一个并发线程之间的计数器。

#### 作用

一个线程等待其他线程任务完成之后才能继续向下执行自己的任务。

向计数器设置一个初始值，任何对象调用这个对象的await()方法都会阻塞，直到这个计数器的计数值被其他线程减为0为止。

#### 场景

​	`有一个任务想要进行下去，但是必须等待其他任务完成之后才能继续向下执行。假如一个想要继续向下执行的任务调用了CountDownLatch的await()方法，其他任务执行自己任务之后调用同一个CountDownLatch的countDown()方法，这个调用await()方法的任务一直会被阻塞，直到计数器的值变为0的时候才会被唤醒后继续向下执行任务。`

#### 案例

​	有三个工人在为老板干活，老板有一个习惯就是当所有工人把活干完之后要统一检查。

```java
package com.gopherasset.juc.tools;
import java.util.Random;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
/**
 * 工人类
 */
public class Worker implements Runnable {
  
    private CountDownLatch downLatch;
    private String name;

    Worker(CountDownLatch downLatch, String name) {
        this.downLatch = downLatch;
        this.name = name;
    }

    @Override
    public void run() {
        long startTime = System.currentTimeMillis();
        this.doWork();
        try {
            TimeUnit.SECONDS.sleep(new Random().nextInt(10));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        this.downLatch.countDown();
      	long endTime = System.currentTimeMillis();
        System.out.println(this.name + "-->活干完了，花了" + (endTime - startTime) + "毫秒");
    }

    private void doWork() {
        System.out.println(this.name + "-->正在干活");
    }
}
```

```java
package com.gopherasset.juc.tools;
import java.util.concurrent.CountDownLatch;
/**
 * 老板类
 */
public class Boss implements Runnable {

    private CountDownLatch downLatch;
  
    Boss(CountDownLatch downLatch) {
        this.downLatch = downLatch;
    }
  
    @Override
    public void run() {
        System.out.println("老板正在等所有的工人干完活......");
        try {
            this.downLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("工人活都干完了，老板开始检查了！");
    }
}
```

```java
package com.gopherasset.juc.tools;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
/**
 * 测试，启动所有任务
 */
public class CountDownLatchDemo {

    public static void main(String[] args) {
        ExecutorService executor = Executors.newCachedThreadPool();
        CountDownLatch countDownLatch = new CountDownLatch(3);
				//定义所有任务
        Worker w1 = new Worker(countDownLatch, "张三");
        Worker w2 = new Worker(countDownLatch, "李四");
        Worker w3 = new Worker(countDownLatch, "王五");
        Boss boss = new Boss(countDownLatch);
				//启动所有的任务
        executor.submit(w1);
        executor.submit(w2);
        executor.submit(w3);
        executor.submit(boss);
        executor.shutdown();
    }
}
```

输出结果：

```
张三-->正在干活
李四-->正在干活
王五-->正在干活
老板正在等所有的工人干完活......
张三-->把活干完了，花了6004毫秒
李四-->把活干完了，花了8005毫秒
王五-->把活干完了，花了8004毫秒
工人活都干完了，老板开始检查了！
```

#### 源码分析

CountDownLatch类中有3个核心方法：CountDownLatch(int count)、await()、countDown()方法

- ##### CountDownLatch(int count)

  ```java
  public CountDownLatch(int count) {
    	if (count < 0) throw new IllegalArgumentException("count < 0");
    	this.sync = new Sync(count);
  }
  ```

  该构造方法根据给定参数count创建一个CountDownLatch，内部创建了一个内部类Sync的实例

  ```java
  private static final class Sync extends AbstractQueuedSynchronizer {
      Sync(int count) {
        setState(count);
      }
    
      int getCount() {
        return getState();
      }
  
      protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
      }
  
      protected boolean tryReleaseShared(int releases) {
        for (;;) {
          int c = getState();
          if (c == 0)
            return false;
          int nextc = c-1;
          if (compareAndSetState(c, nextc))
            return nextc == 0;
        }
      }
  }
  ```

  Sync的构造方法里调用了一个setState方法，因为Sync是继承自AbstractQueueSynchronizer，setState方法就是继承自它的。而AQS里面的state就是内部计数器。

- ##### await()

  ```java
  public void await() throws InterruptedException {
  	sync.acquireSharedInterruptibly(1);
  }
  
  //继承自AbstractQueueSynchronizer的方法
  public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    //如果当前线程中断，就抛出异常
    if (Thread.interrupted()) throw new InterruptedException();
    //尝试获取共享锁，如果可以成功获取就直接返回；获取不到就执行doAcquireSharedInterruptibly方法
    if (tryAcquireShared(arg) < 0)
      doAcquireSharedInterruptibly(arg);
  }
  
  //如果当前内部计数器等于0，返回1，否则返回-1（覆写AbstractQueueSynchronizer的tryAcquireShared方法）
  protected int tryAcquireShared(int acquires) {
    //计数器等于0，表示可以获取共享锁，否则不可以
    return (getState() == 0) ? 1 : -1;
  }
  
  //返回当前内部计数器的值（继承自AbstractQueueSynchronizer）
  protected final int getState() {
    return state;
  }
  
  //该方法使得当前线程一直阻塞，直到当前线程获取到共享锁或者线程被中断才返回
  private void doAcquireSharedInterruptibly(int arg) throws InterruptedException {
    	//根据当前线程创建一个共享模式的Node，并添加到等待队列的尾部
      final Node node = addWaiter(Node.SHARED);
      boolean failed = true;
      try {
        	for (;;) {
            //获取新建节点的前驱节点
          	final Node p = node.predecessor();
            //如果前驱节点是头结点
          	if (p == head) {
              //尝试获取共享锁
            	int r = tryAcquireShared(arg);
              //成功获取到共享锁
            	if (r >= 0) {
                //前驱结点从等待队列中释放；同时使用LockSupport.unpark()唤醒前驱结点的后继节点中的线程
              	setHeadAndPropagate(node, r);
              	p.next = null; // help GC
              	failed = false;
              	return;
            	}
          	}
            
            //当前节点的前驱结点不是头结点，或者不可以获取到共享锁
          	if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
            throw new InterruptedException();
        	}
      	} finally {
        	if (failed) cancelAcquire(node);
      }
  }
  
  private Node addWaiter(Node mode) {
    	//根据当前线程创建一个共享模式的Node
    	Node node = new Node(Thread.currentThread(), mode);
    	Node pred = tail;
    	//如果尾部节点不为空，尾节点的前驱节点指向这个尾节点，同时尾节点指向新节点
    	if (pred != null) {
      	node.prev = pred;
      	if (compareAndSetTail(pred, node)) {
        	pred.next = node;
        	return node;
      	}
    	}
    	//如果尾节点为空(等待队列是空的)，执行enq方法将新节点插入到等待队列的尾部
    	enq(node);
    	return node;
  }
  
  Node(Thread thread, int waitStatus) {
    	this.waitStatus = waitStatus;
    	this.thread = thread;
  }
  
  
  private Node enq(final Node node) {
    	//使用循环插入尾部节点，确保插入成功
    	for (;;) {
      	Node t = tail;
        //尾节点为空(等待队列为空)，新建节点并设置为头节点
      	if (t == null) {
        	if (compareAndSetHead(new Node()))
          	tail = head;
      	} else {
          //否则将节点插入到等待队列的尾部
        	node.prev = t;
        	if (compareAndSetTail(t, node)) {
          	t.next = node;
          	return t;
        	}
      	}
    	}
  }
  
  //获取当前节点的前驱节点
  final Node predecessor() throws NullPointerException {
    	Node p = prev;
    	if (p == null)
      	throw new NullPointerException();
    	else
      	return p;
  }
  
  //判断当前节点里的线程是否需要被阻塞
  private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    	//前驱节点线程的状态
    	int ws = pred.waitStatus;
    	//如果前驱节点线程的状态是SIGNAL，返回true，需要阻塞线程
    	if (ws == Node.SIGNAL)
      	return true;
    	//如果前驱节点线程的状态是CANCELLED，则设置当前节点的前去节点为"原前驱节点的前驱节点"
    	//因为当前节点的前驱节点线程已经被取消了
    	if (ws > 0) {
      	do {
        	node.prev = pred = pred.prev;
      	} while (pred.waitStatus > 0);
      	pred.next = node;
    	} else {
      	//其它状态的都设置前驱节点为SIGNAL状态
      	compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    	}	
    return false;
  }
  
  //通过使用LockSupport.park阻塞当前线程
  //同时返回当前线程是否中断
  private final boolean parkAndCheckInterrupt() {
    	LockSupport.park(this);
    	return Thread.interrupted();
  }
  ```

- ##### countDown()

  ```java
  public void countDown() {
    sync.releaseShared(1);
  }
  
  public final boolean releaseShared(int arg) {
    	//如果内部计数器状态值递减后等于零
    	if (tryReleaseShared(arg)) {
      	//唤醒等待队列节点中的线程
      	doReleaseShared();
      	return true;
    	}
    	return false;
  }
  
  //尝试释放共享锁，就是将内部计数器值减一
  protected boolean tryReleaseShared(int releases) {
    for (;;) {
      //获取内部计数器状态值
      int c = getState();
      if (c == 0) return false;
      //计数器减一
      int nextc = c-1;
      //使用CAS修改state值
      if (compareAndSetState(c, nextc))
        return nextc == 0;
    }
  }
  
  private void doReleaseShared() {
    for (;;) {
      //从头结点开始
      Node h = head;
      //头结点不为空，并且不是尾节点
      if (h != null && h != tail) {
        int ws = h.waitStatus;
        if (ws == Node.SIGNAL) {
          if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
            continue;
          //唤醒阻塞的线程
          unparkSuccessor(h);
        }
        else if (ws == 0 &&
                 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
          continue;
      }
      if (h == head)
        break;
    }
  }
  
  private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
      compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
      s = null;
      for (Node t = tail; t != null && t != node; t = t.prev)
        if (t.waitStatus <= 0)
          s = t;
    }
  
    if (s != null)
      //通过使用LockSupport.unpark唤醒线程
      LockSupport.unpark(s.thread);
  }
  ```

  #### 实战经验

  客户端的一个同步请求查询用户的风险等级，服务端收到请求后会请求多个子系统获取数据，然后使用风险评估规则模型进行风险评估。如果使用单线程去完成这些操作，这个同步请求超时的可能性会很大，因为服务端请求多个子系统是依次排队的，请求子系统获取数据的时间是线性累加的。此可以使用CountDownLatch，让多个线程并发请求多个子系统，当获取到多个子系统数据之后，再进行风险评估，这样请求子系统获取数据的时间就等于最耗时的那个请求的时间，可以大大减少处理时间。



