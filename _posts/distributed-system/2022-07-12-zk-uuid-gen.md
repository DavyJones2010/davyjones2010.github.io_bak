---
layout: post
title: ZK理解再深入-SID生成与snowflake算法
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


## Snowflake ID生成算法
### 概览
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122159430.png)

### 分析

- 41位时间戳, 标识的时间范围是?

1970-01-01 08:00:00 ~ 2039-09-07 23:47:35
> 41位时间截不是存储当前时间的时间截，而是存储时间截的差值（当前时间截 - 开始时间截) 得到的值，这里的的开始时间截，一般是我们的id生成器开始使用的时间，由我们程序来指定的（如下下面程序IdWorker类的startTime属性）。41位的时间截，可以使用69年


- 10位机器id, 标识的机器数量范围是:

1024 台, 5位datacenterId和 5位workerId

- 12位序列号

### 实际场景
RocketMQ, 消息ID是使用snowflake算法生成, 是由客户端产生.
但客户端如何知道自己的工作机器ID? --> 参照： 根据MAC地址生成


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
- zk的算法, 但单host增数量是65535, 适用于长连场景, 即session不会频繁创建, 从而导致后16位递增那么快.
- snowflake算法, 支持单机每毫秒产生 2^12 = 4096 个ID, 适用于创建ID非常频繁的场景.
- 假设, 用zk的算法来生成mq的id:
    - 如果还是机器启动时生成时间戳位, 那单机只能生成 2^16 = 65535 个ID, 之后就必然会重复了!

# 其他总结

1. sid, snowflakeID 本质上都是分布式ID生成, 需要保障几点:
    1. 局部, 全局 唯一
    1. 趋势递增. (这点是UUID无法达到的效果)

# Refs
- [https://www.cnblogs.com/haoxinyue/p/5208136.html](https://www.cnblogs.com/haoxinyue/p/5208136.html)
- [https://www.cnblogs.com/jiangxinlingdu/p/8440413.html](https://www.cnblogs.com/jiangxinlingdu/p/8440413.html)
- [https://xie.infoq.cn/article/ed9b31c014342fd469627d42d](https://xie.infoq.cn/article/ed9b31c014342fd469627d42d)



