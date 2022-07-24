---
layout: post
title: 由ZK的SID/ZXID与snowflake算法引发的ID生成算法探讨
tags: [distributed-system, uuid, snowflake, zookeeper, sid]
lang: zh
---

# 几种生成算法
## zk里的sid生成算法
### 类型

- 一个long类型, 占用64位
- 由ZK服务端生成的.
- 当客户端与ZK服务端建立好TCP连接(或者说应用层连接)之后, 生成.

### 生成规则

1. 获取当前时间(2013-10-04 21:59:42)的毫秒表示：1380895182327 用二进制表示为：

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122157745.png)

2. 将步骤1中的数值左移24位，得到：

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122157302.png)

3. 右移8位：

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122158285.png)

4. 添加机器标识: SID. id 表示配置在myid文件中的值，通常是整数1、2、3等,假设id为2：

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122158613.png)

5. 将步骤3和步骤4得到的两个64位表示的数值进行`或`操作：

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122158687.png)

```java
public static long initializeNextSessionId(long id) {
    long nextSid;
    nextSid = (Time.currentElapsedTime() << 24) >>> 8;
    nextSid = nextSid | (id << 56);
    if (nextSid == EphemeralType.CONTAINER_EPHEMERAL_OWNER) {
        ++nextSid;  // this is an unlikely edge case, but check it just in case
    }
    return nextSid;
}
```

### 详细分析
{8位, 当前主机的myid} {40位, 毫秒时间戳} {16位, 单host递增序列号}
ZK主机启动时, 会把前 48位初始化好, 接下来每次有client链接到该host, 则后16位进行递增.

### 线上样例

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122158988.png)

例如, sid = 0x3 518ae13bc4 16e8 本质上能拆分成:

- 前8位: 0x3, 代表当前myid=3, 是整个ZK集群的第3台服务器.
- 中间40位: 518ae13bc4, 代表初始化时时间戳. 由于移位时截断了最高位1, 因此实际的时间戳是 1518ae13bc4, 转成10进制1449733995460, 按照毫秒数转成时间戳2015-12-10 15:53:15, 可以知道该host启动是在这个时间点.
- 后16位: 16e8, 转成10进制, 5864, 代表是第5864个连接

### 碰撞分析
思考SID产生规则, 是否有碰撞风险?

1. 前8位代表主机位, 最大支持256个主机, 如果集群有上千台服务器, 这样必然会重复, 但这样会导致碰撞么? --> 不太会
    1. 只要能保证前48位不重复, 即可以保证sid不碰撞. 因为后16位是单host粒度递增.
2. 前48位如何碰撞?
    1. 即前8位相同的主机 myid=1 (二进制 00000001) 与 myid=257 (100000001)
    1. 在同一毫秒同时启动, 从而中间40位相同
    1. 而在实际小规模集群情况下, 基本很难产生.
4. 单机上, 后16位如果溢出怎么办?
    1. 即单机上client反复创建session, 超过了2^16=65535, 必然会重复!
    1. 经试验, 发现当后16位满了之后, 会向前边借位. 例如:
        1. 单机上之前的sid: 0x3 764c3db1d3 768e
        1. 后续频繁创建session, 后16位满了, 变成 0x3 764c3db1d4 00e1
    3. 看代码: 无脑地对sid做+1操作. 即使这样, 也不太会导致 中间40位碰撞. (有这个可能, 例如当前session频繁创建, 变成了 0x3 ffffffffff 768e, 接下来服务器在 ffffffffff 这个毫秒点启动, 但当前session未失效, 重新连接上去了.  从而sid从 0x3 ffffffffff 0000开始递增, 有可能重新生成了一个 0x3 ffffffffff 768e 的sid)

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122159565.png)

## zk里的zxid生成算法

### zxid组成
> The zxid has two parts: the epoch and a counter. <br/> 
> In our implementation the zxid is a 64-bit number. <br/>
> We use the high order 32-bits for the epoch and the low order 32-bits for the counter.

64位的long类型, 包含两部分: 前32位代表epoch(即选举次数); 后32位代表counter(即该zk集群中的update操作的次数, 基本是单调递增的).
但实际这样设计是有缺陷的:
- 在实际场景中, quorum一般都是在较为稳定的内网环境下, 不太会因为网络问题导致发生failover选主切换; 因此epoch使用32位, 支持40亿次选举, 没啥必要.
- 而实际counter增长是比较迅猛的, 在支持1000qps的系统中, 50天左右counter就会溢出.
- 而counter溢出会导致发生一次强制选主, 从而把counter清零, 把epoch+1;
- 而在3.3.5版本之前, counter溢出不会选主, 存在bug, 导致zk集群整体不可用. [servers stop serving when lower 32bits of zxid roll over](https://issues.apache.org/jira/browse/ZOOKEEPER-1277)

### zxid生成
- zxid必然是由leader生成, 保证单调递增, 不能由客户端生成.

```java
long epoch = 1;
long counter = 1;
long zxid = epoch << 32 | counter;
// 新的update操作
zxid++;
```

### zxid使用情况
可以使用如下脚本判断后32位使用量, 如果结果>0.8, 代表使用量已经超过80%, 代表有风险.

```shell
echo srvr | nc 127.0.0.1 32188 | awk '/Zxid/{printf "%f\n", and(strtonum($NF),0xffffffff)/2^32}'
```


## Snowflake ID生成算法
### 概览
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122159430.png)

### 分析

- 41位时间戳, 标识的时间范围是?

1970-01-01 08:00:00 ~ 2039-09-07 23:47:35
> 41位时间截不是存储当前时间的时间截，而是存储时间截的差值（当前时间截 - 开始时间截) 得到的值，这里的开始时间截，一般是我们的id生成器开始使用的时间，由我们程序来指定的（如下面程序IdWorker类的startTime属性）。41位的时间截，可以使用69年

- 10位机器id, 标识的机器数量范围是:

1024 台, 5位datacenterId和 5位workerId

- 12位序列号

### 实际场景
RocketMQ, 消息ID是使用snowflake算法生成, 是由客户端产生.
- 客户端如何知道自己的datacenterId?
- 可以通过配置, 也可以如下, 通过本机网卡的MAC生成, 参照：[IdUtil.java](https://github.com/dromara/hutool/blob/a9310c2d305acac617ca656ea6ffc3be6cc48a4c/hutool-core/src/main/java/cn/hutool/core/util/IdUtil.java#L240)

```java
/**
 * 获取数据中心ID<br>
 * 数据中心ID依赖于本地网卡MAC地址。
 * <p>
 * 此算法来自于mybatis-plus#Sequence
 * </p>
 *
 * @param maxDatacenterId 最大的中心ID
 * @return 数据中心ID
 * @since 5.7.3
 */
public static long getDataCenterId(long maxDatacenterId) {
    Assert.isTrue(maxDatacenterId > 0, "maxDatacenterId must be > 0");
    if(maxDatacenterId == Long.MAX_VALUE){
        maxDatacenterId -= 1;
    }
    long id = 1L;
    byte[] mac = null;
    try{
        mac = NetUtil.getLocalHardwareAddress();
    }catch (UtilException ignore){
        // ignore
    }
    if (null != mac) {
        id = ((0x000000FF & (long) mac[mac.length - 2])
                | (0x0000FF00 & (((long) mac[mac.length - 1]) << 8))) >> 6;
        id = id % (maxDatacenterId + 1);
    }

    return id;
}

```

- 客户端如何知道自己的工作机器ID? 
- 根据进程PID与datacenterId生成, 参照：[IdUtil.java](https://github.com/dromara/hutool/blob/a9310c2d305acac617ca656ea6ffc3be6cc48a4c/hutool-core/src/main/java/cn/hutool/core/util/IdUtil.java#L240)

```java
/**
 * 获取机器ID，使用进程ID配合数据中心ID生成<br>
 * 机器依赖于本进程ID或进程名的Hash值。
 *
 * <p>
 * 此算法来自于mybatis-plus#Sequence
 * </p>
 *
 * @param datacenterId 数据中心ID
 * @param maxWorkerId  最大的机器节点ID
 * @return ID
 * @since 5.7.3
 */
public static long getWorkerId(long datacenterId, long maxWorkerId) {
    final StringBuilder mpid = new StringBuilder();
    mpid.append(datacenterId);
    try {
        mpid.append(RuntimeUtil.getPid());
    } catch (UtilException igonre) {
        //ignore
    }
    /*
     * MAC + PID 的 hashcode 获取16个低位
     */
    return (mpid.toString().hashCode() & 0xffff) % (maxWorkerId + 1);
}
```

## aws中资源ID生成算法

### 方案1: Base36
代码如下, 优点是不需要占位符, 可以直接用`ALPHABET`甚至可以修改`ALPHABET`的顺序达到简单加密的效果.
```java
public class Base36Test {

    // 26小写+26大写+10数字=62
    //public static String ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    // 26小写+10数字=36
    public static String ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyz";

    private static String encoding(long num) {
        if (num < 1) {throw new IllegalArgumentException("num must be greater than 0.");}

        StringBuilder sb = new StringBuilder();
        for (; num > 0; num /= 36) {
            sb.append(ALPHABET.charAt((int)(num % 36)));
        }

        return sb.toString();
    }

    private static long decoding(String str) {
        if (str == null || str.trim().length() == 0) {
            throw new IllegalArgumentException("str must not be empty.");
        }

        long result = 0;
        for (int i = 0; i < str.length(); i++) {
            result += ALPHABET.indexOf(str.charAt(i)) * Math.pow(36, i);
        }

        return result;
    }

    public static void main(String[] args) {
        int dcId = 1;
        int izId = 1;

        long l = System.currentTimeMillis();
        String encoding = Base36Test.encoding(l);
        System.out.println(encoding);
        //int idx = (int)(Math.random() * 1024);
        int idx = 1;
        encoding = Base36Test.encoding(idx);
        System.out.println(encoding);

        encoding = Base36Test.encoding(1023);
        System.out.println(encoding);
        encoding = Base36Test.encoding(1024);
        System.out.println(encoding);
    }
}
```

### 方案2: Base64
缺点是会有`=`占位符.


# 时钟回拨问题

- zk里的sid不太会有时钟回拨问题, 是因为时间戳是机器启动的时候生成的. 除非回拨时间特别长, 刚好回拨之后机器又重启了, 拿到了之前那个时间戳. 但考虑到实际场景, 实际不太会发生.
    - 机器重启不会那么频繁, 只会在启动时生成
    - 时钟回拨, 一般都是亚秒级别的回拨
- rocketmq里snowflake, 由于是客户端每次生成时实时获取的时间戳, 因此即使回拨了几毫秒, 在生成ID速度非常快的情况下, 也有可能重复. 如何解决?
    - 比较挫的方案, 关闭ntp
    - 比较好的方案: 当回拨时间小于15ms，就等时间追上来之后继续生成。
    1. 更好的方案, 当时间大于15ms时间我们通过**更换workid**来产生之前都没有产生过的来解决回拨问题。
    1. 最好的方案: 如下修改算法, 可以找2bit位作为时钟回拨位，发现有时钟回拨就将回拨位加1，达到最大位后再从0开始进行循环。

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122159865.png)

# 方案比较
- 相同点: 
  - 都使用long类型, 占用64位
  - 都希望既能尽量减少碰撞, 又能反映递增趋势
  - 组成结构都是: `主机编号+时间戳+递增编号`

- 差异点:
  - 时间戳: 
    - sid时间戳占用40位(完整的应该是41位, 但默认截断了最高位1); 而snowflake时间戳占用41位, 相对时间范围是79年, 绝对时间到2039年. 
    - sid的时间戳, 是zk主机启动起来的时间戳, 之后在下次启动之前, 就不再变化了.
    - snowflake的时间戳, 是client相对于某个时间点的相对时间; 且是生成ID的当前时间, 下一个毫秒这个时间戳会变化.

- 其他
  - zk的算法, 但单host增数量是65535(实际可以借40位, 即上千亿, 但有碰撞风险了), 适用于长连场景, 即session不会频繁创建, 从而导致后16位递增那么快.
  - snowflake算法, 支持单机每毫秒产生 2^12 = 4096 个ID, 适用于创建ID非常频繁的场景.
  - 假设, 用zk的算法来生成snowflake的id:
      - 如果还是机器启动时生成时间戳位, 那单机只能生成 2^16 = 65535 个ID, 之后就只能借时间戳的位了, 可能会重复了!
  - 假设, 用snowflake来生成zk的sid: 
    - 貌似没啥问题.

# 其他总结

1. sid, snowflakeID 本质上都是分布式ID生成, 需要保障几点:
    1. 局部, 全局 唯一
    2. 趋势递增. (这点是UUID无法达到的效果, 因此不使用UUIDGen)

# Refs
- [https://www.cnblogs.com/haoxinyue/p/5208136.html](https://www.cnblogs.com/haoxinyue/p/5208136.html)
- [https://www.cnblogs.com/jiangxinlingdu/p/8440413.html](https://www.cnblogs.com/jiangxinlingdu/p/8440413.html)
- [https://xie.infoq.cn/article/ed9b31c014342fd469627d42d](https://xie.infoq.cn/article/ed9b31c014342fd469627d42d)



