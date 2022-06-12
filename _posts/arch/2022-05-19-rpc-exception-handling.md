---
layout: post
title: RPC调用异常行为与处理方案总结
tags: [rpc, java, exception, learn-from-failure]
lang: zh
---

# 背景
在使用IAcsClient或者dubbo进行rpc调用时, 异常情况总是不可避免的.
这里总结下各种异常, 以及对应的表现形式与处理方式.
# 异常总结

| 错误分类 | 错误细分 | IAcsClient | Dubbo |
| --- | --- | --- | --- |
| 客户端异常 | 客户端网络无连接, 闪断等 |
- IAcsClient 抛出ClientException异常
```java
com.aliyuncs.exceptions.ClientException: SDK.ServerUnreachable : Server unreachable: connection
```
| 
- 抛出RpcException异常: 

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/18490/1654825663340-e0cec9fe-7132-42b4-9ccd-f797fbb29cfa.png#clientId=u7e0ea738-70f5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=421&id=jPcE4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=421&originWidth=1394&originalType=binary&ratio=1&rotation=0&showTitle=false&size=378572&status=done&style=none&taskId=u2098eafb-8903-4080-b4c7-de75c6456ee&title=&width=1394) |
|  | SDK参数缺失错误等 |
- IAcsClient 抛出ClientException异常
```java
com.aliyuncs.exceptions.ClientException: SDK.InvalidProfile : No active profile found.
```
| 
- 不会有该场景, 因为dubbo调用没有SDK
|
| 服务端异常 | 服务端限流, 处理超时 | 
- IAcsClient 抛出ServerException异常
  |  |
  |  | 业务异常, 例如请求参数错误, SPOT价格设置过低等 |
- IAcsClient 抛出ClientException异常
- 注意ServerException是ClientException的子类
  |
- 不抛异常, 返回 PlainResult<Object> isSuccess=false; data=null; message != null

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/18490/1654825572216-6cb392c4-ca80-44c0-9057-5c9a8de3c5b2.png#clientId=u7e0ea738-70f5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=240&id=u1447c812&margin=%5Bobject%20Object%5D&name=image.png&originHeight=240&originWidth=911&originalType=binary&ratio=1&rotation=0&showTitle=false&size=152726&status=done&style=none&taskId=u7751495a-86c7-4df1-bd6b-35d63426195&title=&width=911) |
|  | 服务端未提供该服务 |
- IAcsClient抛出ClientException异常
```java
com.aliyuncs.exceptions.ClientException: InvalidVersion : Specified parameter Version is not valid.
RequestId : xxx

com.aliyuncs.exceptions.ClientException: InvalidAction.NotFound : Specified api is not found, please check your url and method.
RequestId : xxx
```
| 
- 抛出RpcException异常: 
```java
com.alibaba.dubbo.rpc.RpcException: Failed to invoke the method xxx in the service xxx. No provider available for the service xxx from registry xxx
```
|
| 其他异常 |  | IAcsClient正常返回. 返回对象里: 
AcsResponse有requestId, 但返回结果为空, success=false
啥时候会有这种异常?  |  |


# 异常处理
## IAcsClient异常处理
所以针对IAcsClient, 可以按照如下方式进行异常处理:
```java
try {
    ecsInnerClient.getAcsResponse(req);
}
catch (ServerException e) {
     // client调用超时, 被服务端限流等, 都会抛出异常, 走到这里, 需要后续降级到库存缓存实现.
     listResourceResponse.setSuccess(Boolean.FALSE);
     log.error("DescribeResourceAllocation ServerException. request: {}", JSON.toJSONString(req), e);
 } catch (ClientException e) {
     // SPOT价格不满足, 客户端传入参数有错误, 等业务异常, 走到这里, 无需后续降级到库存缓存实现.
     listResourceResponse.setSuccess(Boolean.TRUE);
     log.error("DescribeResourceAllocation ClientException. request: {}", JSON.toJSONString(req), e);
 }
```