---
layout: post
title: MySQL连接池重要参数配置原理研究 
tags: [java, mysql, database, connection-pool, druid]
---

# 重要参数
- initialSize: 初始化时connection数量, 每个connection实际是一条与DB的TCP连接
- maxActive: 最大连接池数量, 对应于maxPoolSize
- minIdle: 最小连接池数量, 对应与minPoolSize
- maxWait: 获取连接等待超时的时间, 具体详解参见: [maxWait参数详解](#maxWait参数详解)
- timeBetweenEvictionRunsMillis: 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
- minEvictableIdleTimeMillis: 配置一个连接在池中最小生存的时间，单位是毫秒
- maxEvictableIdleTimeMillis:  配置一个连接在池中最大生存的时间，单位是毫秒
- maxPoolPreparedStatementPerConnectionSize: PS Cache.暂时不管

## maxWait参数详解

- 参数释义: [https://www.jianshu.com/p/1ff2bd62dd45](https://www.jianshu.com/p/1ff2bd62dd45)
1. 即客户端从连接池中获取connection的最大等待时长
1. 如果不配置, 默认为-1, 即客户端会一直等待. 这样显然是不合适的.
1. client从连接池中获取connection等待超时, 错误信息如下:
```
Caused by: com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 3000, active 4, maxActive 4, creating 0
    at com.alibaba.druid.pool.DruidDataSource.getConnectionInternal(DruidDataSource.java:1722)
    at com.alibaba.druid.pool.DruidDataSource.getConnectionDirect(DruidDataSource.java:1402)
    at com.alibaba.druid.filter.FilterChainImpl.dataSource_connect(FilterChainImpl.java:5059)
    at com.alibaba.druid.filter.logging.LogFilter.dataSource_getConnection(LogFilter.java:886)
    at com.alibaba.druid.filter.FilterChainImpl.dataSource_connect(FilterChainImpl.java:5055)
    at com.alibaba.druid.filter.FilterAdapter.dataSource_getConnection(FilterAdapter.java:2756)
    at com.alibaba.druid.filter.FilterChainImpl.dataSource_connect(FilterChainImpl.java:5055)
    at com.alibaba.druid.filter.stat.StatFilter.dataSource_getConnection(StatFilter.java:680)
    at com.alibaba.druid.filter.FilterChainImpl.dataSource_connect(FilterChainImpl.java:5055)
    at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1380)
    at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1372)
    at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:109)
    at org.springframework.jdbc.datasource.DataSourceTransactionManager.doBegin(DataSourceTransactionManager.java:262)
    ... 11 more
```

- maxWait参数的坑: [https://tech.youzan.com/shu-ju-ku-lian-jie-chi-pei-zhi/](https://tech.youzan.com/shu-ju-ku-lian-jie-chi-pei-zhi/)
- 扩展阅读: Druid锁的公平模式, [https://github.com/alibaba/druid/wiki/Druid%E9%94%81%E7%9A%84%E5%85%AC%E5%B9%B3%E6%A8%A1%E5%BC%8F%E9%97%AE%E9%A2%98](https://github.com/alibaba/druid/wiki/Druid%E9%94%81%E7%9A%84%E5%85%AC%E5%B9%B3%E6%A8%A1%E5%BC%8F%E9%97%AE%E9%A2%98)

# 实战
## 项目1配置样例

- initialSize: 100
- maxActive: 200
- minIdle: 100
- maxWait: 6000

注意, 这个是单机的连接配置, 考虑到分布式环境:

- 初始化情况下, db承受的连接数是 initialSize*nodeCount = 100*81 = 8100
- 压力最大情况下, db承受的连接数是 initialSize*nodeCount = 200*81 = 16200


## 项目2配置项

- initialSize: 未配置
- maxActive: 300
- minIdle: 30
- maxWait: 60000

# 思考

1. 当客户端并发度超过最大连接数时, 会怎样?
    1. 连接池中连接数, 从min --> max 的增长策略是怎样的?
    2. 当增长到max之后, 请求线程会等待maxWait, 如果还是获取不到, 则抛错.
2. 当MySQL服务端连接数超过机器能承受的最大连接数时, 会怎样?
3. MySQL服务端一般能支持的最大连接数是多少?
    1. 取决于物理机/虚拟机的配置.
4. 什么是MySQL的session? 与connection是啥关系?
    1. [https://www.zhihu.com/question/30325800](https://www.zhihu.com/question/30325800)
    2. 和网站的一个session差不多吧，只不过session是把key放在cookie里面，数据库连接是把key放在客户端的library的内存里（比如.Net Sql Client)。对MS SQL来说，这个连接的协议叫TDS，底下可以走多种传输层协议，比如tcpip，也可以named pipe。而MySQL就又有自己的协议。
    3. com.taobao.pamirs.transaction.TBConnection#queryDBSessionID 只有oracle有sessionId. 
