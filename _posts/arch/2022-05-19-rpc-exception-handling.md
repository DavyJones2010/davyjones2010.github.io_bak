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

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122208285.png) |
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

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207122209444.png) |
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