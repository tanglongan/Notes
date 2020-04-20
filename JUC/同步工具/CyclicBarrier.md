CyclicBarrier是多个线程任务之间的一个循环屏障

#### 1、作用

会让所有线程都等待完成之后才会进行下一步行动任务

#### 2、原理

CyclicBarrier在内部定义了一个Lock对象，每当一个线程调用await()方法，将拦截的线程数减1，然后判断剩余线程数是否等于初始值parties，如果不是就进入Lock对象的条件队列等待；如果是，就调用BarrierAction对象的Runnable方法，然后将锁的条件队列中的所有线程放入锁的等待队列中，这些线程会依次获取锁、释放锁。

#### 3、CyclicBarrier与CountDownLatch的区别

- CountDownLatch是线程组之间的等待，CyclicBarrier是线程组内线程之间的等待
- CountDownLatch是减数计数方式，Cyclicbarrier是加数计数的方式
- CountDownLatch计数为0无法重置，CyclicBarrier计数到达初始值后可重置
- CountDownLatch不可以重用，CyclicBarrier可以重用

#### 4、案例

所有游戏玩家进入游戏之后，比赛才开始，否则就相互等待。

```java
package com.gopherasset.juc.tools;
import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;
/**
 * 游戏玩家
 */
public class Player implements Runnable {

    private final String name;
    private final CyclicBarrier cyclicBarrier;

    Player(String name, CyclicBarrier cyclicBarrier) {
        this.name = name;
        this.cyclicBarrier = cyclicBarrier;
    }

    @Override
    public void run() {
        try {
            TimeUnit.SECONDS.sleep(new Random().nextInt(10));
            System.out.println(this.name + "=====>已经进入游戏，等待中");
            cyclicBarrier.await();
            System.out.println(this.name + "=====>看到了所有玩家都到了，大战开始");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
```

```java
package com.gopherasset.juc.tools;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
public class CyclicBarrierDemo {

    public static void main(String[] args) {
        List<String> heros = Arrays.asList("东邪黄药师", "西毒欧阳锋", "南帝段智兴", "北丐洪七公");
        ExecutorService executor = Executors.newFixedThreadPool(heros.size());
        final CyclicBarrier cyclicBarrier = new CyclicBarrier(heros.size());

        for (String hero : heros) {
            executor.submit(new Player(hero, cyclicBarrier));
        }
        executor.shutdown();

    }
    
}
```

输出结果：

```
北丐洪七公=====>已经进入游戏，等待中
东邪黄药师=====>已经进入游戏，等待中
西毒欧阳锋=====>已经进入游戏，等待中
南帝段智兴=====>已经进入游戏，等待中
南帝段智兴=====>看到了所有玩家都到了，大战开始
北丐洪七公=====>看到了所有玩家都到了，大战开始
西毒欧阳锋=====>看到了所有玩家都到了，大战开始
东邪黄药师=====>看到了所有玩家都到了，大战开始
```

#### 5、源码分析

```java
public class CyclicBarrier {
  
		//静态内部类，记录当前屏障是否破坏
    private static class Generation {
        boolean broken = false;
    }
  
		//还没有达到屏障的线程数量
    private int count;
  
  	//构造方法1：parties是需要拦截的线程数
    public CyclicBarrier(int parties) {
        this(parties, null);
    }
  
  	//构造方法2：barrierAction是为了处理更复杂的场景，当线程到达共同屏障的时候优先执行barrierAction
		public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
		
  	//唤醒所有等待中的线程，同时重置线程数为初始线程数量，创建新的屏障代供下一次循环等待使用
    private void nextGeneration() {
        trip.signalAll();
        count = parties;
        generation = new Generation();
    }
  
		//破坏公共屏障
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }
  
  	//获取需要等待的线程数量
		public int getParties() {
        return parties;
    }
  
		//屏障是否破坏
    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }
		//重置线程数量并初始化下一个屏障代
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier(); 
            nextGeneration();
        } finally {
            lock.unlock();
        }
    }
  
		//获取已经到达屏障，并且等待状态的线程数量
    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
  	
  	//不带等待超时时间
  	public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe);
        }
    }
		
  	//具有等待超时时间的方法
    public int await(long timeout, TimeUnit unit) throws Exception {
        return dowait(true, unit.toNanos(timeout));
    }
  
    /**
     * 核心同步方法
     */
    private int dowait(boolean timed, long nanos) throws Exception{
      final ReentrantLock lock = this.lock;
      //通过Reentrant获取独占锁
      lock.lock();
      try {
        //判断屏障是否破坏，如果破坏，则抛出BrokenBarrierException异常
        final Generation g = generation;
        if (g.broken) throw new BrokenBarrierException();
        //当前线程是否被interrupted，如果被中断，则breakBarrier破坏屏障
        if (Thread.interrupted()) {
          breakBarrier();
          throw new InterruptedException();
        }
        //记录当前屏障等待线程数量
        int index = --count;
        //最后一个到达屏障的线程
        if (index == 0) {  // tripped
          boolean ranAction = false;
          try {
            final Runnable command = barrierCommand;
            if (command != null)
              command.run();
            ranAction = true;
            //执行下一个Generation
            nextGeneration();
            return 0;
          } finally {
            //如果barrierCammond执行失败，执行屏障破坏处理
            if (!ranAction)
              breakBarrier();
          }
        }
        //如果当前线程不是最后一个线程
        for (;;) {
          try {
            if (!timed) //调用Condition的await()方法阻塞
              trip.await();
            else if (nanos > 0L)//调用Condition的awaitNanos()方法阻塞
              nanos = trip.awaitNanos(nanos);
          } catch (InterruptedException ie) {
            //如当前线程被中断，判断是否其他线程已破坏屏障，若没有则进行屏障破坏，并抛出异常；否则再次中断当前线程
            if (g == generation && ! g.broken) {
              breakBarrier();
              throw ie;
            } else {
              Thread.currentThread().interrupt();
            }
          }

          if (g.broken) throw new BrokenBarrierException();
          if (g != generation) return index;
          if (timed && nanos <= 0L) {
            breakBarrier();
            throw new TimeoutException();
          }
        }
      } finally {
        //5、通过ReentrantLock释放独占锁。
        lock.unlock();
      }
  }
}
```

核心思想就是：`先判断当前线程是否是最后一个到达屏障的，如果是最后一个达到，就先判断barrierCommand是否为空，不为空执行barrierCommand任务，接着执行nextGeneration方法。在nextGeneration方法中通过Condition的signalAll()方法唤醒其他阻塞的线程继续执行。`

