---
layout: post
title: 记一次Maven传递依赖失效问题排查
tags: [learn-from-failure, java, mdc, npe]
lang: zh
---

# 背景
在test-scheduler项目中, 代码按照maven模块(module)组织, 如下:
```xml
- test-scheduler
---- shared-common
---- resource-manage
---- persistence
```
其中
resource-manage模块在pom中依赖了shared-common模块

# 问题分析
## 错误信息是怎样的?
在运行resource-manage模块的ut时, 报错信息如下:
```java
Caused by: java.lang.NoClassDefFoundError: com/xxx/xxx/manager/ManagerListener
	at com.xxx.xxx.client.MetaPushConsumer.start(MetaPushConsumer.java:338)
	at com.xxx.CommonMetaPushConsumer.start(CommonMetaPushConsumer.java:23)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeCustomInitMethod(AbstractAutowireCapableBeanFactory.java:1930)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1872)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1800)
	... 42 more
Caused by: java.lang.ClassNotFoundException: com.xxx.xxx.manager.ManagerListener
	at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:355)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
	... 51 more
```
## 丢失的类是在哪里的?
经分析, com.xxx.xxx.manager.ManagerListener 类是在如下GAV中引入:
<dependency>
<groupId>com.xxx.xxx</groupId>  
<artifactId>diamond-client</artifactId>  
<version>3.9.2</version>  
</dependency>

## 理论上传递依赖是怎样的?
经分析, diamond-client 是由 shared-common 模块直接依赖的.
由于maven有传递依赖的特性, 因此依赖图如下:
```
resource-manage --依赖--> shared-common --依赖--> diamond-client
```
因此resource-manage应该依赖到了diamond-client, 但实际在intellij里查看依赖图, 发现并没有传递依赖进来!

## 为什么传递依赖没有生效?
经分析发现, 在 test-scheduler 的pom里, 对 shared-common 的依赖进行了裁剪, 将 diamond-client exclude 出去了.


而 resource-manage 等模块, 都是继承了 test-scheduler 的父pom. 因此resource-manange声明依赖了全量的 shared-common 模块如下:

但实际生效的仍是 test-scheduler pom中声明的 shared-common 依赖, 即排除掉了 diamond-client 依赖.


## 问题解决
至此解决问题就很简单了. 在 test-scheduler 的pom中, 对 shared-common 依赖的声明, 删除掉如下片段即可:
<exclusion>
<artifactId>diamond-client</artifactId>
<groupId>com.xxx.diamond</groupId>
</exclusion>
这样就解决问题了

# 扩展研究
## 传递依赖失效的其他场景
