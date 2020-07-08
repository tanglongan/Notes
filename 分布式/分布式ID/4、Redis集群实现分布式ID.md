### Redis分布式ID实现原理

#### Redis分布式ID组成部分

- 毫秒级时间
- Redis集群的第几个节点
- 每一个Redis节点在每一毫秒的自增序列值

假设系统是64位的，然后整数的第一位必须是0，所以最大整数就是63位的，即111111111111111111111111111111111111111111111111111111111111111在这里我们分出41位作为毫秒，然后12位作为Redis节点的数量，最后10位作为Redis在每一毫秒的自增序列值。

- 41位二进制11111111111111111111111111111111111111111转换成10进制的毫秒就是2199023255551，然后我们把 2199023255551转换成时间就是2039-09-07，也就是说可以用20年的。
- 12位二进制111111111111作为Redis节点个数，即最多可以支持4095个redis节点。
- 10位二进制1111111111作为每一个Redis节点可以每一毫秒最多可以生成1013个不重复的值。

#### Java版本的实现

下面的1565165536640L是一个毫秒值，然后我们Redis节点设置为53个，然后设置了两个不同的自增序列值，分别是1和1023。下面的结果展示的就是在1565165536640L这一毫秒里面，53号redis节点生成了两个不同的分布式ID值。

```java
import java.text.SimpleDateFormat;
import java.util.Date;

public class Test {
    public static void main(String[] args) {
        long buildId = buildId(1565165536640L, 53, 1);
        System.out.println("分布式id是："+buildId);
        long buildIdLast = buildId(1565165536640L, 53, 1023);
        System.out.println("分布式id是："+buildIdLast);
    }
    public static long buildId(long miliSecond, long shardId, long seq) {
        return (miliSecond << (12 + 10)) + (shardId << 10) + seq;
    }
}
```

输出结果如下

```
分布式id是：6564780070991352833
分布式id是：6564780070991353855
```

这个结果的不太符合分布式ID的要求，没有可读性，使用下面的方式来获取这个分布式ID的生成毫秒时间值

```java
import java.text.SimpleDateFormat;
import java.util.Date;
public class Test {
    public static void main(String[] args) {
        long buildId = buildId(1565165536640L, 53, 1);
        parseId(buildId);
        long buildIdLast = buildId(1565165536640L, 53, 1023);
        parseId(buildIdLast);
    }
    public static long buildId(long miliSecond, long shardId, long seq) {
        return (miliSecond << (12 + 10)) + (shardId << 10) + seq;
    }
    public static void parseId(long id) {
        long miliSecond = id >>> 22;
        long shardId = (id & (0xFFF << 10)) >> 10;
        System.err.println("分布式id-"+id+"生成的时间是："+new SimpleDateFormat("yyyy-MM-dd").format(new Date(miliSecond)));
        System.err.println("分布式id-"+id+"在第"+shardId+"号redis节点生成");
    }
}
```



