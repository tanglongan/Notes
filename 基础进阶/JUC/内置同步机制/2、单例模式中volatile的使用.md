使用“双重锁检测”实现单例的时候，关于volatile关键字引发的一些探索和思考。

## 不使用volatile会有什么问题？

一个不使用volatile的双重锁检验单例模式大概长这样：

```java
public class Singleton {
    
    private static Singleton instance; // 不使用volatile关键字
    
    // 双重锁检验
    public static Singleton getInstance() {
        if (instance == null) { // 第7行
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // 第10行
                }
            }
        }
        return instance;
    }
}
```

这个代码会有什么问题？我们知道，对一个锁的解锁happens-before随后对这个锁的加锁。粗略一看，上述代码是没有太大问题的。加锁操作并不能保证同步区内的代码不会发生重排序。对于第10行，是可能会被JVM分解和重排序的，也就是说：

```java
instance = new Singleton(); // 第10行

// 可以分解为以下三个步骤
1 memory=allocate();	// 分配内存 相当于c的malloc
2 ctorInstanc(memory) 	// 初始化对象
3 s=memory 				// 设置s指向刚分配的地址

// 上述三个步骤可能会被重排序为 1-3-2，即
1 memory=allocate();	// 分配内存 相当于c的malloc
3 s=memory 				//设置s指向刚分配的地址
2 ctorInstanc(memory) 	//初始化对象
```

而一旦假设发生了这样的重排序，比如线程A在第10行执行了步骤1和步骤3，但是步骤2还没有执行完。这个时候线程A执行到了第7行，它会判定instance不为空，然后直接返回了一个未初始化完成的instance！

## volatile如何解决这个问题？

针对上述问题，在Java 5 以后，JMM模型允许我们使用volatile关键字禁止这样的重排序。对于JMM的happens-before规则，即对一个volatile修饰的变量的写操作，happens-before随后对这个变量的读操作。所以我们可以在声明instance的时候，给它加上volatile关键字。

```java
public class Singleton {

    private static volatile Singleton instance; // 使用volatile关键字
    
    // 双重锁检验
    public static Singleton getInstance() {
        if (instance == null) { // 第7行
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // 第10行
                }
            }
        }
        return instance;
    }
}
```

OK，问题似乎解决了。

