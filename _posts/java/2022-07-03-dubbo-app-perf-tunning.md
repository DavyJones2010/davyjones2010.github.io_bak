---
layout: post
title: 记一次Dubbo应用性能优化
subtitle: 记一次Dubbo应用性能优化
tags: [dubbo, java, perf-tunning, redis]
---

# 背景
新的应用以dubbo接口对外暴露调度能力。
在上线前需要整体压测，确认下系统能力与瓶颈。

## 目标
目标QPS: 500+
目标RT: 100ms-

## 整体架构
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207042216549.png)

## 应用框架
- SpringBoot 2.5.6
- Druid 1.1.22

## 机器配置清单
- ECS规格： ecs.n4.2xlarge
- 2台8C16G 独享型
- CPU: 2.5 GHz主频的Intel ® Xeon ®处理器
- JDK 1.8
- 网络： 1.2Gbps，即至少支持100MB/s的内网带宽

## 中间件配置清单

### Redis
- 规格：8C32GB
- 支持的最大连接数量：30000
- Host到Redis的Ping延时在1ms以内

### DB/RDS
- 规格：mysql.x4.4xlarge.2 32C128GB
- MySQL 5.7
- 支持的最大连接数量：20000
- Host到DB的Ping延时在1ms以内


# 压测记录
## 第一轮压测

### 压测结果
- 压到140QPS时， RT已经飙升到8s+；

### 其他指标
- cpu: 已经基本跑满了，利用率780+%；
- mem: 内存占用量很低，总体稳定在34%左右。
- 网络： 总体入流量在10Mbps以下，出流量1Mbps以下，远远没有打到带宽上限。
- 磁盘： 没有频繁的磁盘IO
- Java: 没有发生频繁的GC。

### 原因分析
根据应用日志，发现有如下几个问题：
- Redis访问
  - 单次schedule()请求，会连续串行调用40+次的redis get接口。
  - 这些串行的get调用，加起来平均耗时在6s+
- 其他问题：
  - apache.util中Pair类型序列化问题

### Redis访问速度慢问题分析
#### 是否是服务端问题？
查看了Redis服务端整体性能情况：
- 出口帶寬爲100MB，當時只利用了10%左右
- CPU Usage 14% 左右
- 連接數：最大支持30000，當時只有3個穩定的連接
- QPS: 高峯期QPS爲6000，而實例的max爲24w，使用率僅爲2.5%，遠遠沒有達到限流閾值

#### 是否是客户端问题？
- 網絡流量：10MB左右，遠遠沒有打滿。
- Load: 很低，0.2以下

#### 問題定位
在這種情況下，一般就只能懷疑是client連接問題。類似HttpClient，客戶端請求都在排隊等着獲取connection。

- 方案1： 增大連接池配置，增大最大連接數量。
- 方案2： 減少請求次數


### 优化方案
- Redis优化：
  - 采用了方案2： 使用Redis的mget，而不是单个get。以减少网络交互
  - 为啥没采用方案1？ 
> 因为 SpringBoot 1.5.x 版本默认使用Jedis作为redis client，比较好调整connection数量等。<br/>
> 但 SpringBoot 2.x 版本默认使用了Lettuce作为redis client，本身基于Netty的异步IO方式实现，与Jedis不同，本身就支持单个连接被多个线程同时访问。
> 所以官方不建议配置连接池。

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207042214249.png)

- 其他问题优化
  - 自己重新实现了一版Pair类型的序列化、反序列化方式

## 第二轮压测

### 压测结果
- 在250QPS时，RT基本稳定在100ms以下
- 压到300QPS时，RT突然飙升到3s+
- cpu利用率较低
- 内存占用量很低，总体稳定在34%左右
- 没有发生GC

### 原因分析
根据日志打点，很容易找到当时的瓶颈就在ram-client调用鉴权服务鉴权上。
- RAM接口问题：


### 优化方案
- RAM Client优化：
  - 增大client的max-connections数量，增大到300。

> 这里有个插曲，当增大client的max-connections数量到300之后，重新进行了一次压测，发现QPS并没有提升，RT的瓶颈仍然在访问RAM上！ <br/>
> 后续看了ram-client的发布记录，发现属于ram-client的一个bug。<br/>
> ram-client使用了http-client作为内部实现。<br/>
> 当前版本中，增大max-connections数量，只是增大了httpclient全局支持的connection数量，而单个host支持的connection数量仍然是默认值，即8. <br/>
> 由于访问ram的endpoint就是单个host，因此相当于该配置没有生效！ <br/>
> 后续升级了ram-client版本，新版本实现里，将max-connections值同时赋给了max-connections-per-host，因此相当于max-connections-per-host也被调大为300了<br/>
> 关于max-connections与max-connections-per-host参数的详细解释，参见： [异步HttpClient使用Netty作为SocketChannel的Provider](https://davyjones2010.github.io/2022-03-15-http-client-netty/)


## 第三轮压测
### 压测结果
- 200QPS, 80ms RT
- 300QPS，2000ms RT

- 结果与上轮相比，并没有显著提升。
- 根据日志分析，ramclient耗时已经趋于平稳，平均10ms以下，调度耗时又上来了。

### 其他指标
- cpu: 利用率较高，780%+
- mem: 内存占用量很低，总体稳定在34%左右。
- 网络： 总体流量在10Mbps，远远没有打到带宽上限。
- 磁盘： 没有频繁的磁盘IO
- Java: 没有发生频繁的GC。


### 原因分析
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207042215103.png)
根据当时的线程堆栈分析，占用CPU量最大的就是8个lettuce线程，几乎把CPU时间片占满了。
同时搜索了相关问题 ，发现[不是个例](https://gitter.im/lettuce-io/Lobby?at=5de8f93446397c721c8ed8ed)。
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207042215636.png)

### 优化方案
- Redis:
  - 使用Jedis作为Client，并且调大Jedis对应的连接池数量

```properties
spring.redis.client-type=jedis
spring.redis.jedis.pool.max-active=100
spring.redis.jedis.pool.max-idle=16
spring.redis.jedis.pool.max-wait=10000
```



## 第四轮压测
### 压测结果
- 300QPS，100ms RT
- 350QPS，200ms RT
- 400QPS，2000ms RT

### 其他指标
- cpu: 利用率较高，780%+
- mem: 内存占用量很低，总体稳定在34%左右。
- 网络： 总体流量在10Mbps，远远没有打到带宽上限。
- 磁盘： 没有频繁的磁盘IO
- Java: 没有发生频繁的GC。

### 原因分析
打印出了当时的线程栈信息，进行了详细的分析如下：

| 操作           | 线程数量  |
|------| ----  |
| 獲取Druid連接    | 442 |
| 歸還Druid連接    | 164 |
| 業務線程中Bind操作  | 107 |
| Log4j打印      | 77 |
| Jedis執行      | 2 |

- 大部分线程卡在Druid数据库连接池归还连接、获取连接的搶鎖阶段（lock()）
- 小部分线程由于bind()异步线程池队列满，导致占用了业务线程
- 最后一部分线程卡（blocked）在log4j打印日志


#### Druid连接池问题
與 [记一次Dubbo线程耗尽的问题-druid数据库连接池突发性能](https://blog.csdn.net/beFocused/article/details/108533137) 文中的問題一模一樣。


#### 异步线程队列满问题
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207042216269.png)

隊列默認長度爲200，在200的長度滿了之後，使用的是`ThreadPoolExecutor.CallerRunsPolicy`, 即不再在異步線程裏執行，而是在caller線程中執行。
從而導致業務線程壓力進一步增大。

#### Log4j日志问题
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207042216085.png)
也是比較經典的問題，使用Log2j的異步logger即可解決。

### 优化方案
- Druid优化： 
  - 增大连接池中initial与max的连接数量
  - 使用非公平锁，而不是默认的公平锁。具体参见[有赞的压测记录](https://mp.weixin.qq.com/s/RaiU9_ioWHvomZLLKuSuGw)。
```properties
spring.datasource.druid.maxActive=600
spring.datasource.druid.initialSize=300
spring.datasource.druid.maxWait=6000
spring.datasource.druid.minIdle=300
spring.datasource.druid.poolPreparedStatements=true
spring.datasource.druid.maxOpenPreparedStatements=20
spring.datasource.druid.useUnfairLock=true
```

- bind线程池优化：
  - 增大队列大小，从200增大到20000
- log4j优化：
  - 使用log4j2的asyncLogger。

## 第五轮压测
### 压测结果
- 在600QPS时，RT基本稳定在40ms以下。
- 由于每次调用接口，都会调用一次ram鉴权接口，ram整体限制单个租户需要小于600qps；也担心把ram接口打挂或者被ram限流，因此不再继续压测。
- 按照其他指标推测，<mark> 理论上可以打到1000QPS+，即单机500QPS+ </mark>
- 终于达标了！~ 撒花庆祝~~~

### 其他指标
- cpu: 利用率较低，400%左右
- mem: 内存占用量很低，总体稳定在34%左右。
- 网络： 总体流量在10Mbps，远远没有打到带宽上限。
- 磁盘： 没有频繁的磁盘IO
- Java: 没有发生频繁的GC。

# 优化总结



