---
layout: post
title: 记一次由于remoteCache类型不兼容修改导致的异常问题
tags: [learn-from-failure, java, cache]
lang: zh
---

# 背景
进行Redis缓存升级改造, 缓存访问层统一进行了修改: 
修改前代码如下, 按照Java原生序列化方式: 
```java
// 写缓存, Long类型
cache.put("key", 123L);
// 读缓存, Long类型
(Long)cache.get("key");
```
修改后代码如下, 统一按照String类型进行序列化/反序列化:
```java
// 写缓存, 转成String类型
cache.put("key", "123");
// 读缓存, 获取String类型
(String)cache.get("key");
```

# 问题
线上发布过程中报出异常, `ClassCastException`, Long类型无法被cast成String类型.
经分析, 由于是集群的环境, 发布过程中, 针对同一个"key", 
- 未布机器写入的还是Long类型
- 已发布的机器读取该"key"对应的value, 尝试强转成String, 
反之亦然, 也会有 String类型无法被cast成Long类型的报错. 


# 总结
在分布式环境下, 对remote cache的修改, 一定要注意兼容性:
- 不止新版本旧版本的兼容性
- 更要考虑到发布过程中新老机器, 针对同一个key的类型的兼容性.