---
layout: post
title: 记一次Maven传递依赖失效问题排查
tags: [learn-from-failure, maven, transitive dependency]
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
### scope问题
1. 由于本问题中是在UT运行时发生的CNFE(ClassNotFoundException), 因此基本可以排除由于scope的原因. 
2. 但如果是在运行时发生了CNFE, 则需要看下是否使用了<scope>test/provided/system</scope>等标签

### optional问题
- 如果optional为false, 则代表引用该pom的项目, 不会间接依赖到该GAV
```xml
<dependency>
    <groupId>net.biancheng.www</groupId>
    <artifactId>X</artifactId>
    <version>1.0-SNAPSHOT</version>
    <!--设置可选依赖  -->
    <optional>true</optional>
</dependency>
```

- 关于 optional 元素及可选依赖说明如下：
1. 可选依赖用来控制当前依赖是否向下传递成为间接依赖；
2. optional 默认值为 false，表示可以向下传递称为间接依赖；
3. 若 optional 元素取值为 true，则表示当前依赖不能向下传递成为间接依赖。
4. 仅适用于 项目的单纯依赖关系，不适合 父子工程。假设A工程是 parent，那么A工程即便加上了 optinal，子项目也将继承父工程的所有依赖关系。

- 在什么场景下, 会把<optional>true</optional>设置为true?
假设一个关于数据库持久化的项目(Project C), 为了适配更多类型的数据库持久化设计，比如 Mysql 持久化设计(Project A) 和 Oracle 持久化设计(Project B)，
- 当我们的项目(Project D) 要用的 Project C 的持久化设计，不可能既引入 mysql 驱动又引入 oracle 驱动吧，所以我们要显式指定一个，就是这个道理了




# Refs
- [Maven依赖传递,排除依赖和可选依赖](https://www.cnblogs.com/cy0628/p/15034450.html#:~:text=Maven%20%E7%9A%84%E4%BE%9D%E8%B5%96%E4%BC%A0%E9%80%92%E6%9C%BA%E5%88%B6,%E4%B8%8A%E7%AE%80%E5%8C%96POM%20%E7%9A%84%E9%85%8D%E7%BD%AE%E3%80%82)
- [Maven optional关键字透彻图解](https://juejin.cn/post/6844903987322290189)

