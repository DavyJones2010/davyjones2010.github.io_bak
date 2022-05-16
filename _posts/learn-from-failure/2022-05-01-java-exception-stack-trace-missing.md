---
layout: post
title: 记一次由于JVM优化导致异常堆栈丢失的问题
tags: [learn-from-failure, java, stacktrace]
lang: zh
---

# 背景
线上某次故障排查, 发现线上有异常, 但无堆栈信息. 
实际查看打印异常的地方, 确实是会打印出堆栈的. 那么堆栈信息为啥会丢呢? 
```java
java.lang.ClassCastException: null
```

# 原因排查
JVM默认会优化掉相同堆栈的信息: 
[java.lang.ClassCastException with null message and cause](https://stackoverflow.com/questions/40502576/java-lang-classcastexception-with-null-message-and-cause)


# 总结

1. 如 [“java.lang.ClassCastException: null”原因解决](https://blog.csdn.net/lisheng19870305/article/details/106361875), 针对如下类型的异常, JVM默认都会优化: 
```java
NullPointerException
ArrayIndexOutOfBoundsException
ClassCastException
ArrayIndexOutOfBoundsException
ArrayStoreException
ArithmeticException
```
2. 以后错误日志最好保留时间长一点, 这样才能定位出异常首次出现的位置, 才能看到完整的堆栈信息. 
3. 或者可以增加: `-XX:-OmitStackTraceInFastThrow` 参数来防止JVM优化. 

