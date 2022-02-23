---
layout: post
title: 一些日常踩的小坑笔记
tags: [learn-from-failure, java, mdc, npe]
lang: zh
---

# Java SPI机制未生效, ServiceLoader获取不到实现类
---
最终定位到原因, 是因为接口实现的文件需要放在 `main/resources/META-INF/services/MyInterface` 中
而实际文件放到了: `main/resources/META-INF.services/MyInterface` 中了.

这里比较坑的一点是, 在IntelliJ IDEA中, 会默认flatten package, 即将目录路径层级用`.`连接, 因此
`main/resources/META-INF/services/MyInterface` 与 `main/resources/META-INF.services/MyInterface`
两个目录看起来是一样的, 如下(根本分不清楚到底是文件夹名字叫`codecontest.bloomfilter`还是文件夹是`codecontest/bloomfilter`): 
![img.png](img.png)

随便搜了下, 发现大家踩同样坑的也不少, 目前看貌似没有好的办法避免, 记住有这个坑, 后续引以为戒吧!
- [intellij idea包路径和文件夹目录的坑](https://blog.csdn.net/tszxlzc/article/details/65938891)

# Java SPI机制创建的实例无法被SpringContext管理问题
---
## 问题代码样例:
```java
/**
 * 加载实现
 */
public class MyServiceLoader {
    public static void main(String[] args) {
        ServiceLoader<MyInterface> interfaces = ServiceLoader.load(MyInterface.class);
        for (MyInterface anInterface : interfaces) {
            anInterface.sayHello();
        }
    }
}

```

```java
/**
 * 实现类
 */
@Component
public class MyImpl implements MyInterface {
    @Autowired
    MyService myService;

    @Override
    public void sayHello() {
        myService.doSomething();
    }
}
```

## 原因分析
发现在`MyImpl#sayHello`会抛出NPE, 即`myService`对象为null.
很显而易见, 因为 `MyImpl` 实例是从ServiceLoader中new出来的, 而不是由Spring容器创建的, 因此自身就脱离了SpringContext的管理.
因此`@Component`这个annotation本质是不生效的, `myService` 也不会被Spring自动注入进去. 

## 解决方案
### 方案1: 手动注入 
手动获取Context, 手动注入: 
```java
/**
 * 实现类
 */
public class MyImpl implements MyInterface {
    // SpringBeanTool代码参见 https://www.jianshu.com/p/7fc4358b4e36
    MyService myService = SpringBeanTool.getBean(MyService.class);

    @Override
    public void sayHello() {
        myService.doSomething();
    }
}
```
### 方案1的问题思考:
但一个疑问是如果MyImpl类中`myService`属性的初始化早于SpringContext的初始化 
那么 `SpringBeanTool.getBean(MyService.class);` 必然返回的是null(因为SpringContext尚未初始化完成).
实际运行测试, 发现不会返回null. 那么实际的先后顺序是怎样的? 是否有可能返回null?

### 方案2: 使用SpringBoot的SPI?? 
```java
// TODO: 这个先留个坑, 待后续研究. 
``` 

## Refs
- [Spring Boot 获取上下文环境](https://www.jianshu.com/p/7fc4358b4e36)
- [springboot-starter中的SPI 机制](https://juejin.cn/post/6844903890173837326)

# 底层cache从Tair切换到Redis遇到的坑
---
本质上是由于 `@Cachable` 默认使用了Jackson序列化方式, 而cache中的对象是匿名内部类, 从而反序列化失败.
> 内部类实例需要其外部类实例对象来进行实例化，而Jackson在反序列化时无法创建其外部类实例对象
## Refs
- [一次反序列化内部类导致的问题排查过程](https://juejin.cn/post/6844904170403659784) 

