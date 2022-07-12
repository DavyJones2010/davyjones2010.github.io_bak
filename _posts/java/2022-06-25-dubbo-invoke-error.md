---
layout: post
title: 记一次Dubbo服务调用异常排查
tags: [java, dubbo]
---

# 问题描述
- 应用拓扑: systemA ---- dubbo 调用 ----> systemB
- systemB应用在已有的dubbo接口类里, 新增加了一个方法methodA.
- systemA在调用该methodA时, 抛错. 错误信息如下:

# 排查步骤
## 0x00 排查provider侧
确认provider侧:

1. 服务是否正常可以执行? > 通过telnet localhost, 手动invoke确定是OK的.
2. 接口类&接口方法是否正常注册在registry上? > 通过查看dubbo registry, 发现接口类与接口方法是正常注册的.

## 0x01 排查consumer侧
确认consumer侧:

1. 是否正确依赖到了provider?
    1. 通过查看 tcp ESTABLISHED 连接确定长连接已经建立.
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122201347.png)
    2. or 查看registry下发的 dubbo config 文件?  --> TODO: 暂时没找到, 待分析.
2. 请求是否正确路由到了provider的host?  // 由于systemB有多套环境, 怀疑是请求路由到了非目标环境.
    1. 动态分析: 查看 dubbo 请求日志 --> 没找到, 待进一步查看.
    2. 静态分析: 查看 diamond 配置 + 相关代码, 是不是路由代码有误?

## 0x02 真实错误原因

- provider侧与consumer侧都没有问题, 路由也没有问题, 但为啥会报错?
- 最终在某位同学的提醒下, 查看了consumer侧的error日志, 发现了详细的错误堆栈信息如下(原来是序列化失败):

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122201975.png)

## 0x03 原因探讨
### 为啥dubbo调用参数必须要是Serializable的?
因为dubbo rpc, 默认使用hessian2序列化方式. 
[而hessian2针对java Object类型参数, 使用的是默认的Java序列化方式](https://blog.csdn.net/liyong1028826685/article/details/117308356). 
而Java序列化则要求Object必须`implements Serializable`接口.


### 为啥在本地telnet invoke的时候没有序列化失败错误?
因为在本地telnet invoke的时候, 默认使用的是JSON序列化方式!

### 如何查看当前接口使用的序列化方式?

1. 查看registry, 如果没有指定, 则是默认的hessian2:

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122202268.png)

### 如何指定序列化方式?
[https://dubbo.apache.org/zh/docs/references/xml/dubbo-protocol/](https://dubbo.apache.org/zh/docs/references/xml/dubbo-protocol/)
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122202940.png)

```shell
<dubbo:protocol id="dubbo-m" threadpool="dubboThreadPool" name="dubbo"  port="${pbs.dubbo.protocol.default.port}"  threads="150" 
serialization="json" />

<dubbo:protocol id="dubbo-h" threadpool="dubboThreadPool" name="dubbo"  port="${pbs.dubbo.protocol.high.port}"  threads="150" serialization="fastjson" />
```

# 收获
## 正确的问题排查步骤

1. 排查问题时, 优先在报错的host(不用管是consumer/provider)上, 查看error等更详细的堆栈日志.

## 走的一些弯路

1. provider侧排查时, invoke 方法名称写错, 导致误以为是provider没有把新的接口方法暴露出去. 误导了排查方向. 如下:

正确的: `invoke com.xxx.xxx.XXXQueryService.queryHistory("")`
错误的: `invoke com.xxx.xxx.XXXQueryService.describeHistory("")`
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122202508.png)
[如何快速查看某个host的某个interface有没有把某个method暴露出去](#dubbo如何查到某台host中某个接口提供的所有方法)

2. consumer侧排查时, 由于systemA的dubbo console的端口与systemB的dubbo console 端口不一致, 也查了半天.
[如何快速查到dubbo console端口号?](#dubbo如何查看console的端口)

## Dubbo最佳实践
### dubbo按照端口进行线程池隔离
#### 隔离原理
单个host可以开启多个dubbo端口, 每个端口由独立的线程池处理, 把svc按照端口进行分组.
可以防止低优先级接口占用线程导致dubbo线程池满, 从而影响高优先级接口.
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122202434.png)
#### 隔离配置样例

```xml
<dubbo:registry id="coreRegistry" address="${pbs.dubbo.registry.address}" default="true" />
<!-- 核心接口 -->
<dubbo:protocol id="dubbo-core" threadpool="dubboThreadPool" name="dubbo"  port="${pbs.dubbo.protocol.core.port}"  threads="200" />
<!-- 中优先级接口线程池(默认，为了最大保持兼容) -->
<dubbo:protocol id="dubbo-m" threadpool="dubboThreadPool" name="dubbo"  port="${pbs.dubbo.protocol.default.port}"  threads="150" />
<!-- 默认provider使用dubbo-m,http-dubbo协议 -->
<dubbo:provider protocol="dubbo-m" filter="-sentinel.dubbo.provider.filter" delay="-1" />

<!-- 使用dubbo-core -->
<dubbo:service interface="com.xxx.TestService"
protocol="dubbo-core" version="${pbs.service.version}" ref="testService" retries="0" timeout="30000"/>
<!-- 使用dubbo-core -->
<dubbo:service interface="com.xxx.SampleService"
protocol="dubbo-core" version="${pbs.service.version}" ref="sampleService" retries="0" timeout="30000"/>
```

### dubbo默认端口号配置原则
如上, `pbs.dubbo.protocol.default.port=-1`配置项, 按照dubbo文档说明, 应该是随机分配一个端口. 但发现systemB实际启动后, 却开启了 20880 端口. 这是为啥?
#### 官方文档
根据 [https://dubbo.apache.org/zh/docs/references/xml/dubbo-protocol/](https://dubbo.apache.org/zh/docs/references/xml/dubbo-protocol/)  文档说明,  `分配的端口在协议缺省端口的基础上增长`
但实际语焉不详, dubbo协议默认端口是20880,  那么如果配置 `port=-1`, 则端口是 `20880` or `20881`??

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122203219.png)

#### 源代码分析
核心代码: `com.alibaba.dubbo.config.ServiceConfig#doExportUrlsFor1Protocol`, 可以得知是从 20880 开始(包括20880).
这也就解释了, 为啥 `pbs.dubbo.protocol.default.port=-1`配置项, 实际对应的开启端口号是 20880 啦.
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122203404.png)
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122203097.png)


### dubbo多端口telnet原则
如上, 暴露了 20880~20884 5个端口, 用于线程池隔离.
实际:
1. 每个端口, 都是启动了一个NettyServer
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122203903.png)
2. 因此每个端口, 都可以telnet上去.
3. telnet每个端口执行 ls 列举出来本机暴露的dubbo服务都是一样的(会把所有其他端口暴露的服务也都枚举出来). 不会因为telnet 20880, 就只列举出暴露在20880端口的服务. 参见: `com.alibaba.dubbo.rpc.protocol.dubbo.telnet.ListTelnetHandler`
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122204305.png)

### dubbo版本号隔离
用来区分多套预发环境.
但实际看了下, 多套预发环境provider的版本号都是1.0.0, 而不是通过版本号来进行隔离的. 那么consumer实际怎么做的路由?
// TODO:  核心代码附录
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122204657.png)

### dubbo序列化

- 使用console手动invoke, 用的是json序列化.
- 而consumer真正调用时用的是hessian2序列化, 因此针对这种序列化差异导致的错误问题, 是无法验证到的. 需要编码测试的时候注意.

### dubbo ls/invoke
如何查看当前host下依赖的所有dubbo服务?

1. 通过telnet localhost xxxx 进入dubbo管控页面, 执行 `ls`命令, 只能列出自身作为provider对外暴露的所有服务.
1. 而无法列出自身host作为consumer依赖到的所有服务.  // 查看了dubbo文档, 目前没有办法通过telnet查看到.

### dubbo如何查到某台host中某个接口提供的所有方法
#### 方案1 telnet查看
`ls -l com.xxx.SampleService`
telnet localhost  20880  telnet localhost  20881 telnet localhost  20882 都能登上去, 实际方法列表也相同.  参见: [dubbo多端口telnet原则](#th8th)
#### 方案2 registry查看
在registry中查询接口, 选定host查看:
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122206622.png)

### dubbo如何查看console的端口
#### 方案1: 通过查看应用配置文件
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122207808.png)
> 这种方式最直接, 也最有效. 缺点是紧急排查问题时, 如果配置项经过了一层有一层的自动替换, 不一定能很快找出实际配置值是啥.

#### 方案2: 通过查看TCP连接

1. 查看进程号PID(注意看清楚进程的USER是不是当前用户)
```shell
ps aux | fgrep ${appName}
```

2. 查看进程占用的TCP在Listen状态的端口(如果上一步获取到的PID不是当前用户的, 则需要su到PID对应的用户, 或者如下用sudo)
```shell
sudo lsof -i -P| fgrep ${PID} | fgrep LISTEN
```

3. 看到如下, 一般dubbo端口号是从20880开始的(包括20880)

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122207567.png)

但如果应用配置的dubbo端口很奇怪, 不符合惯例, 那这里就只能一个一个端口来telnet试试了.
> 这种方式不算非常直接, 但优点是可以不用去翻找代码, 翻找配置项. 在端口数很少的情况下, 简单有效.


# Refs

- [Dubbo Serialize 层：多种序列化算法，总有一款适合你](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Dubbo%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB%E4%B8%8E%E5%AE%9E%E6%88%98-%E5%AE%8C/16%20%20Dubbo%20Serialize%20%E5%B1%82%EF%BC%9A%E5%A4%9A%E7%A7%8D%E5%BA%8F%E5%88%97%E5%8C%96%E7%AE%97%E6%B3%95%EF%BC%8C%E6%80%BB%E6%9C%89%E4%B8%80%E6%AC%BE%E9%80%82%E5%90%88%E4%BD%A0.md)




