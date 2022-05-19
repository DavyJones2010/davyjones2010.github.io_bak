---
layout: post
title: 记SpringBootCache使用的问题总结
tags: [learn-from-failure, java, springboot, spring-cache]
lang: zh
---

# @Cacheable 无参方法
有些无参方法, 也需要被SpringBoot @Cacheable 进行cache, 如下代码样例:  
```java
@Cachable(value = "my_cache", unless="#result == null || #result.size() <= 0")
public List<String> listAllUids() {
    return Lists.newArrayList("a", "b");
}
```

有如下需要注意的点:
- 可以不写 `key=xxx`, 这样默认的key就是`SimpleType []`
- SpringBoot官方建议[使用方法名作为key](https://docs.spring.io/spring/docs/3.1.x/spring-framework-reference/html/cache.html), 如下: 

```java
@Cachable(value = "my_cache", unless="#result == null || #result.size() <= 0", key = "#root.method.name")
```

- 也可以hardcode一个key, 这样更易于查找, 如下: 

```java
@Cachable(value = "my_cache", unless="#result == null || #result.size() <= 0", key="'my_key'")
```

-  要注意key里的常量, <font color='red'>一定要用单引号括起来</font>, 否则会[报错如下](https://stackoverflow.com/questions/33383366/cacheble-annotation-on-no-parameter-method): 
```java
org.springframework.expression.spel.SpelEvaluationException: EL1008E:(pos 0): Property or field my_key cannot be found on object of type 'org.springframework.cache.interceptor.CacheExpressionRootObject' - maybe not public?
```