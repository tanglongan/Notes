## 1、分布式锁实现的要点

为了实现分布式锁，需要确保锁同时满足以下4个条件：

- 互斥性。任意时刻只能有一个客户端能持有锁。
- 不会发生死锁。即使一个客户端持有锁期间发生崩溃而没有释放锁，也需要保证其他客户端能够获取到锁。
- 加锁和解锁必须是同一个客户端，客户端不能把其他客户端的加的锁给释放了。
- 容错性。只要大部分Redis节点运行正常，客户端就可以进行加锁和解锁的操作。

## 2、基于Redis实现分布式锁的错误姿势

### 2.1、加锁错误姿势

首先为了保证互斥性和不会发生死锁，在加锁操作的时候，需要使用setnx命令来保证互斥性，即只有一个客户端可以加锁成功。为了保证不发生死锁，需要给锁加一个过期时间，这样即使持有锁的客户端发生崩溃也不会影响其他客户端持有锁。为了保证这2点，有一个初步的错误实现方式如下：

```java
/**
 * 实现加锁的错误姿势
 */
public static void wrongGetLock1(Jedis jedis, String lockKey, String requestId, int expireTime) {
    Long result = jedis.setnx(lockKey, requestId);
    if (result == 1) {
        // 若在这里程序突然崩溃，则无法设置过期时间，将发生死锁
        jedis.expire(lockKey, expireTime);
    }
}
```

错误实现的原因：setnx和expire两个redis操作不具备原子性。如果程序在执行setnx之后突然崩溃导致没有设置锁的过期时间，从而就导致死锁了。因为这个客户端自己崩溃了不会主动释放锁，同时它持有的锁也不会被其他客户端释放掉，最终导致其他客户端不能获取到锁，就会一直阻塞。`所以针对该代码实现应该保证setnx和expire操作具备原子性。`因此改进之后的错误姿势2的具体实现如下

```java
/**
 * 实现加锁的错误姿势2
 */
public static boolean wrongGetLock2(Jedis jedis, String lockKey, int expireTime) {
    long expires = System.currentTimeMillis() + expireTime;
    String expiresStr = String.valueOf(expires);
    // 如果当前锁不存在，返回加锁成功
    if (jedis.setnx(lockKey, expiresStr) == 1) {
        return true;
    }

    // 如果锁存在，获取锁的过期时间
    String currentValueStr = jedis.get(lockKey);
    if (currentValueStr != null && Long.parseLong(currentValueStr) < System.currentTimeMillis()) {
        // 锁已过期，获取上一个锁的过期时间，并设置现在锁的过期时间
        String oldValueStr = jedis.getSet(lockKey, expiresStr);
        if (oldValueStr != null && oldValueStr.equals(currentValueStr)) {
            // 考虑多线程并发的情况，只有一个线程的设置值和当前值相同，它才有权利加锁
            return true;
        }
    }
    // 其他情况，一律返回加锁失败
    return false;
}
```

这个加锁操作咋一看没有毛病对吧。那以上这段代码的问题毛病出在哪里呢？

- 由于客户端自己生成过期时间，所以需要强制分布式环境下所有客户端的时间保持一致。
- 当锁过期的时候，如果多个客户端同时执行jredis.getSet()方法。虽然最终之后一个客户端执行成功，但是这个客户端的过期时间有可能被其他客户端覆盖。不具备加锁解锁操作是由同一个客户端操作的原则。`解决上面这段代码的方式就是每个客户端加锁添加一个标记来确保加锁和解锁操作是同一个客户端的操作。`

### 2.2、解锁的错误姿势

```java
/**
 * 解锁错误姿势1
 * 最简单直接的解锁方式，这种不先判断拥有者而直接解锁的方式，会导致任何客户端都可以随时解锁。即使这把锁不是它上锁的。
 */
public static void wrongReleaseLock1(Jedis jedis, String lockKey) {
    jedis.del(lockKey);
}
```

```java
/**
 * 解锁错误姿势2
 */
public static void wrongReleaseLock2(Jedis jedis, String lockKey, String requestId) {
    // 判断加锁与解锁是不是同一个客户端
    if (requestId.equals(jedis.get(lockKey))) {
        // 若在此时，这把锁突然不是这个客户端的，则会误解锁
        jedis.del(lockKey);
    }
}
```

既然错误姿势1中没有判断锁的拥有者，那姿势2中判断了拥有者，那错误原因又在哪里呢？答案又是原子性上面。因为判断和删除不是一个原子性操作。在并发的时候很可能发生解除了别的客户端加的锁。具体场景有：客户端A加锁，一段时间之后客户端A进行解锁操作时，在执行jedis.del()之前，锁突然过期了，此时客户端B尝试加锁成功，然后客户端A再执行del方法，则客户端A将客户端B的锁给解除了。从而不也不满足加锁和解锁必须是同一个客户端特性。`解决思路就是需要保证GET和DEL操作在一个事务中进行，保证其原子性。`

## 3、Redis实现分布式锁的正确姿势

### 3.1、加锁的正确方式

- 使用setnx命令保证互斥性
- 设置锁的过期时间，避免死锁
- setnx和设置锁过期时间的操作需要保证原子性，避免在setnx加锁成功之后在设置过期时间之前客户端发生崩溃导致死锁。
- 加锁的value值为唯一标识。加锁成功后需要把这个唯一标识返回给客户端进行解锁操作时使用。可以使用uuid作为标识。

### 3.2、解锁的正确方式

- 需要拿加锁的唯一标识作为解锁依据，从而保证加锁和解锁是同一个客户端的操作。
- 解锁操作需要比较唯一标识是否相等，相等再执行删除操作。比较和删除这2个操作可以采用Lua脚本方式实现。

### 3.3、正确的实现代码

```java
public interface DistributedLock {
    /** 获取锁 */
    String acquire();
    /** 释放锁 */
    boolean release(String indentifier);
}
```

```java
@Slf4j
public class RedisDistributedLock implements DistributedLock{

    private static final Long   RELEASE_SUCCESS = 1L;

    private Jedis jedis;			//redis 客户端
    private String lockKey;			//分布式锁的键值
    int expireTime = 10 * 1000; 	//锁的超时时间 10s
    int acquireTimeout  = 1 * 1000;	//获取锁超时时间

    public RedisDistributedLock(Jedis jedis, String lockKey) {
        this.jedis = jedis;
        this.lockKey = lockKey;
    }

    public RedisDistributedLock(Jedis jedis,String lockKey, int acquireTimeout) {
        this.jedis = jedis;
        this.lockKey = lockKey;
        this.acquireTimeout = acquireTimeout;
    }

    public RedisDistributedLock(Jedis jedis, String lockKey, int acquireTimeout, int expireTime) {
        this.jedis = jedis;
        this.lockKey = lockKey;
        this.acquireTimeout = acquireTimeout;
        this.expireTime = expireTime;
    }

    @Override
    public String acquire() {
        try {
            // 计算锁的超时时间，超过这个时间则放弃获取锁
            long end = System.currentTimeMillis() + acquireTimeout;
            // 生成客户端的唯一标识
            String requireToken = UUID.randomUUID().toString();
            while (System.currentTimeMillis() < end) {
                String result = jedis.set(lockKey,requireToken,"NX","PX",expireTime);
                if ("OK".equals(result)) {
                    return requireToken;
                }
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        } catch (Exception e) {
            log.error("acquire lock due to error", e);
        }
        return null;
    }

    @Override
    public boolean release(String identify) {
　　　 if(identify == null){
            return false;
        }

        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1])"+
                         "else return 0 end";
        Object result = new Object();
        try {
            result = jedis.eval(script,Collections.singletonList(lockKey),Collections.singletonList(identify));
        if (1L.equals(result)) {
            log.info("解锁成功, 客户端ID:{}", identify);
            return true;
        }}catch (Exception e){
            log.error("解锁失败：",e);
        }finally {
            if(jedis != null){
                jedis.close();
            }
        }

        log.info("解锁失败, 客户端ID:{}, 解锁结果:{}", identify, result);
        return false;
    }
}
```

下面就以秒杀库存数量为场景，测试下上面实现的分布式锁的效果。具体测试代码如下：

```java
public class RedisDistributedLockTest {
    static int n = 500;
    public static void secskill() {
        System.out.println(--n);
    }

    public static void main(String[] args) {
        Runnable runnable = () -> {
            RedisDistributedLock lock = null;
            String unLockIdentify = null;
            try {
                Jedis conn = new Jedis("127.0.0.1",6379);
                lock = new RedisDistributedLock(conn, "lock");
                unLockIdentify = lock.acquire();
                System.out.println(Thread.currentThread().getName() + "正在运行");
                secskill();
            } finally {
                if (lock != null) {
                    lock.release(unLockIdentify);
                }
            }
        };

        for (int i = 0; i < 10; i++) {
            Thread t = new Thread(runnable);
            t.start();
        }
    }
}
```

