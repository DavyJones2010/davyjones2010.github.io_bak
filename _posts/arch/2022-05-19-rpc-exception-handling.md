---
layout: post
title: RPC调用异常行为与处理方案总结
tags: [rpc, java, exception, learn-from-failure]
lang: zh
---

| 错误分类  | 错误细分  | IAcsClient  | Dubbo  |
|---|---|---|---|
| 客户端异常 | 客户端网络无连接, 闪断等  | TODO  |   |
| 客户端异常 | SDK参数缺失, 校验异常等  |  TODO |   |
| 服务端异常 | 系统异常(例如服务端限流, 处理超时等)  | IAcsClient抛出ServerException异常 |   |
| 服务端异常 | 业务处理异常(例如服务端参数校验错误, 数量Quota被限制等)  | IAcsClient抛出ClientException异常  |   |


