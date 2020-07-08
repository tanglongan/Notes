### 1、UUID的生成策略

`UUID的方式能生成一串唯一随机32位长度数据，它是无序的一串数据。按照开放软件基金会(OSF)制定的标准计算，UUID的生成用到了以太网卡地址、纳秒级时间、芯片ID码和许多可能的数字。`UUID的底层是由一组32位数的16进制数字构成，是故 UUID 理论上的总数为16^32 = 2^128，每纳秒产生1百万个 UUID，要花100亿年才会将所有 UUID 用完，所以这足够我们的使用了，也能够保证唯一性。

### 2、UUID的格式

UUID 的十六个八位字节被表示为 32个十六进制数字，以连字号分隔的五组来显示形式为 8-4-4-4-12，总共有 36个字符（即32个英数字母和四个连字号）

```properties
123e4567-e89b-12d3-a456-426655440000
xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
```

### 3、Java中的UUID生成

```java
import java.util.UUID;
public class Test {
    public static void main(String[] args) {
        String uuid= UUID.randomUUID().toString().replace("-", "").toLowerCase();
        System.out.println("UUID的值是:"+uuid);
    }
}
```

### 4、UUID是否适合做分布式ID？

如果需求只要唯一性，那么UUID是合适的。但是在复杂场景中，UUID其实是不能作为分布式ID，原因如下：

> - 首先分布式id一般都会作为主键，但是安装mysql官方推荐主键要尽量越短越好，UUID每一个都很长，所以不是很推荐
> - 既然分布式id是主键，主键是包含索引的，然后MySQL的索引是通过B+树来实现的，每一次新的UUID数据的插入，为了查询的优化，都会对索引底层的b+树进行修改，因为UUID数据是无序的，所以每一次UUID数据的插入都会对主键字段的B+树进行很大的修改，会有性能问题。
> - 信息不安全：基于MAC地址生成UUID的算法可能会造成MAC地址泄露，这个漏洞曾被用于寻找梅丽莎病毒的制作者位置。

