---
layout: post
title: SpringBoot基于Interface的Annotation在AOP中如何生效I: 基础概念
tags: [java, spring, spring-boot, aop, interface]
---

# 背景
SpringBoot项目代码里, 新增了一个 `@Perf` 的annotation, 希望增加该annotation的方法, 能自动打印出方法的执行耗时.
实际代码结构中, 使用了FilterChain模式, 会有多个Filter(例如10+个)实现同一个接口. 
因此想着把 `@Perf` 打在接口层面, 希望所有实现类都能继承该annotation, 从而不用在每个子类的Filter方法里重复打上annotation. 
但发现实际没有生效.

# 原因分析
在stackoverflow里, 类似的问题也是一堆,
总结下来原因是如下两个:  
1. method 上的 annotation 没有被继承下来. 参见如下继承原则详解. 
2. spring 在对类/对象进行进行增强(weaving)时, 校验bean是否需要被增强 `@Around("@annotation(Perf)")`, 只会校验bean对应的class上的annotation, 而不会向上回溯父类/接口上是否被打了该annotation.  

## 1. Java Annotation的继承原则
### 1.1 Annotation继承原则: 
> Java中作用在方法层面的annotation不能被继承
> Java中作用在interface层面的annotation不能被继承
> Java中打在class层面的annotation默认不能被继承, 但如果annotation被打上@Inherited标签, 则可以被子类继承

- 原则参见: <a href="http://www.eclipse.org/aspectj/doc/released/adk15notebook/annotations.html#annotation-inheritance">annotation-inheritance</a>
- 详细演示测试代码参见: [edu.xmu.kunlun.headfirst.spring.annotation.AnnotationTest](https://github.com/DavyJones2010/head-first-spring/blob/feature/20220603_annotation_on_interface/src/test/java/edu/xmu/kunlun/headfirst/spring/annotation/AnnotationTest.java)

### 1.2 为啥会有这种继承原则?
上述原则看起来比较复杂, 且不可思议. 如果死记硬背, 效率太低. 需要从设计者角度来考虑下为啥不直接都能继承就好?
想清楚原因, 这些原则自然就明白了. 自己查询了stackoverflow, 根本原因是<font color='red'>为了避免歧义</font>. 详细场景分析如下: 
1. 为啥作用在方法层面的annotation不能被继承?
因为java interface是多继承的
原因样例参见: [edu.xmu.kunlun.headfirst.spring.annotation2.Sub2](https://github.com/DavyJones2010/head-first-spring/blob/feature/20220603_annotation_on_interface/src/main/java/edu/xmu/kunlun/headfirst/spring/annotation2/Sub2.java) 

2. 为啥作用在interface层面的annotation不能被继承?
因为java interface是多实现的.
如果class同时实现了多个interface, 每个interface都实现同一个annotation(但对应annotation的参数不同), 那么当前class使用的该是哪个annotation(with参数)?
这种场景下, 在OOP中术语叫做 [diamond inheritance](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem)

3. 为啥作用在class层面的annotation可以被继承?
因为java class是单继承的. 所以不会有interface的问题.

### 1.3 问题总结
因此可以得知, Filter接口的 filter() 方法上的 annotation 没有被继承下来.

## 2. Spring AOP 增强原理与原则
### 2.1 Spring AOP 中几个重要概念
- AOP proxy: an object created by the AOP framework in order to implement the aspect contracts (advise method executions and so on). In the Spring Framework, an AOP proxy will be a JDK dynamic proxy or a CGLIB proxy.
- Weaving: linking aspects with other application types or objects to create an advised object. This can be done at compile time (using the AspectJ compiler, for example), load time, or at runtime. Spring AOP, like other pure Java AOP frameworks, performs weaving at runtime.
所以, 最关键问题是由于 Spring Boot Weaving 时根据annotation没有weave上去, 而跟使用何种类型的 AOP Proxy 无关! 


### 2.2 Spring AOP Weaving 核心代码
weaving的几种类型:
- compile time weaving: 在编译期生成增强类, 例如使用 AspectJ compiler  
- run time (or load time) weaving: 在运行期, 实时地校验是否符合条件, 

### 2.3 Spring AOP Weaving 核心逻辑
Spring AOP Weaving是load time weaving, 即在Spring容器构建完所有的bean之后, 完成容器启动前执行的, 使用的是aspectj weaver

- 核心代码参见: `org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean`
- 而针对该问题, 即判断该Aspect能否应用在该类型上, 核心逻辑如下:
`org.springframework.aop.support.AopUtils#canApply(org.springframework.aop.Pointcut, java.lang.Class<?>, boolean)`
即判断 `@Perf` 能否应用在 `edu.xmu.kunlun.headfirst.spring.service.impl.FilterA` 对象上:  
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206051422714.png)
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206051424707.png)

- 而看实际实现如下, 是直接根据targetClass来判断, 确实没有向上继续寻找父类/接口层面是否有声明 `@Perf`, 因此返回是false: 
`org.springframework.aop.aspectj.AspectJExpressionPointcut#matches(java.lang.reflect.Method, java.lang.Class<?>, boolean)`
 ![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206051438400.png)

- 再来思考下, 为啥不向上继续寻找? 这个就又回到了 第一个问题, Java Annotation的继承原则; 如果向上寻找, 有可能在同一层次找到多个同样的annotation, 但with不同的参数. 在这种情况下, Spring根本就不知道该以哪个为准了! 如下述例子: 

```java
public @interface Perf {
    boolean print() default false;
}

@Perf(print = true)
public interface Filter {
    
}

@Perf(print = false)
public interface AnotherFilter {

}

// 如果Spring Weaving会向上(父类/接口)寻找, 那么到底以哪个为准? print=false or print=true??
public class FilterA implements Filter, AnotherFilter {
}
```

# 解决方案

再回到最初的问题, 如何能解决该问题?

## 方案1:
将`@Perf`打在各个子类的实现里, 缺点是: 非常麻烦, 后续有其他子类, 都需要记得打上annotation.

## 方案2:  
不使用annotation作为pointcut的匹配条件, 而采用如下表达式:
`@Around("execution(public * edu.xmu.kunlun.headfirst.spring.service.Filter+.doFilter(..))")`

- 优点: `edu.xmu.kunlun.headfirst.spring.service.Filter`的所有子类的`doFilter`都会被自动增强.
- 缺点: 如果有其他接口例如`edu.xmu.kunlun.headfirst.spring.service.Weighter`的所有实现需要被增强, 则需要修改pointcut表达式, 不方便. 
- [代码样例 edu.xmu.kunlun.headfirst.spring.aspect.PerfAspect2](https://github.com/DavyJones2010/head-first-spring/blob/feature/20220603_annotation_on_interface/src/main/java/edu/xmu/kunlun/headfirst/spring/aspect/PerfAspect2.java)

## 最终方案:
如上分析Spring AOP相关源码, 没有找到更好的方案了. 这也是避免后续踩坑的最佳实践了. 
个人使用 `方案1` 作为了最终方案.

# 扩展思考
## Spring接口层面支持的 @Transactional annotation
如[接口上注解 AOP 拦截不到场景兼容实例演示](https://cloud.tencent.com/developer/article/1832305) 文中所言:
Spring本身支持的 `@Transactional` annotation, 是可以打在interface上, 然后子类就自动实现了transaction相关增强的功能. 
那么Spring具体是怎么实现的呢?
- Spring官方的解答如下: 

> The Spring team recommends that you annotate only concrete classes (and methods of concrete classes) with the @Transactional annotation, as opposed to annotating interfaces. 
> You certainly can place the @Transactional annotation on an interface (or an interface method), 
> but this works only as you would expect it to if you use interface-based proxies. 
> The fact that Java annotations are not inherited from interfaces means that, 
> if you use class-based proxies (proxy-target-class="true") or the weaving-based aspect (mode="aspectj"), 
> the transaction settings are not recognized by the proxying and weaving infrastructure, 
> and the object is not wrapped in a transactional proxy.

实际上, 也引发了很多讨论: 
- [Where should I put @Transactional annotation: at an interface definition or at an implementing class?](https://stackoverflow.com/questions/3120143/where-should-i-put-transactional-annotation-at-an-interface-definition-or-at-a)
- 文中推荐或者说Spring团队推荐 `@Transactional` 最好要打在`concrete classes`上, 而不要打在interface上.
- 因为 <mark><font color='red'>如果使用的如果不是JDK Based Proxy(or interface-based proxy), 则该annotation是不生效的!!!</font></mark>
- 所以<mark>如果要用在interface上, 一定要知道当前使用的proxy是哪种.</mark> 
- 而在实际上, 本身Spring/SpringBoot的默认proxy方式一直在变, 我们很难弄明确清楚(或者要弄清楚需要费很大劲儿). 下文会详细说明.

## SpringBoot中默认代理方式
根据 [AOP in Spring Boot, is it a JDK dynamic proxy or a Cglib dynamic proxy?](https://www.springcloud.io/post/2022-01/springboot-aop/#gsc.tab=0) 文中说明, 
Spring默认的AOP Proxy与SpringBoot默认的是有区别的 
### Spring默认AOP Proxy
- If the proxy object implements the interface, then use the JDK dynamic proxy, otherwise it is the Cglib dynamic proxy.
- If the proxy object does not implement an interface, then it is a direct Cglib dynamic proxy.

### SpringBoot默认AOP Proxy
又根据SpringBoot 1.0 与 SpringBoot 2.0 版本有所区分: 
- SpringBoot 1.0 AOP Proxy原则: 默认用 JDK proxy
> If the developer has set spring.aop.proxy-target-class to false, then the JDK proxy is used.
If the developer has spring.aop.proxy-target-class set to true, then the Cglib proxy is used.
If the developer did not configure the spring.aop.proxy-target-class property in the first place, then the JDK proxy is used.

- SpringBoot 2.0 AOP Proxy原则: 默认用 Cglib
> If the developer has set spring.aop.proxy-target-class to false, then the JDK proxy is used.
If the developer has spring.aop.proxy-target-class set to true, then the Cglib proxy is used.
If the developer did not configure the spring.aop.proxy-target-class property in the first place, then the Cglib proxy is used.


### 其他好玩儿的
- 之前线上也有过一次故障, 如下`nestedHystrixWrappedGetCurrentThreadId`方法调用, 调用到的 `hystrixWrappedGetCurrentThreadId` 则实际没有被增强.
- 是因为调用 `hystrixWrappedGetCurrentThreadId` 实际是由 被代理对象(target) 调用的, 而不是由 代理对象(proxy) 调用的. 具体样例参见: [edu.xmu.kunlun.headfirst.spring.service.impl.UserSvcTest](https://github.com/DavyJones2010/head-first-spring/blob/feature/20220603_annotation_on_interface/src/test/java/edu/xmu/kunlun/headfirst/spring/service/impl/UserSvcTest.java)

```java
@HystrixWrapper(commandGroupKey = "blog")
public long hystrixWrappedGetCurrentThreadId() {
  return getCurrentThreadId();
}
 
public long nestedHystrixWrappedGetCurrentThreadId() {
  return hystrixWrappedGetCurrentThreadId();
}
```

## 最佳实践
- `@Transactional` <mark>不要打在接口上, 一定要打在实现类上!!</mark>
- 显式声明`spring.aop.proxy-target-class=true`, <mark>让Spring/SpringBoot项目统一都用cglib作为proxy方式. </mark> `As a reminder, to always use CGLIB, just set the “spring.aop.proxy-target-class” property to true.`

> Before using CGLIB, 
> ensure your codebase always uses pre-existing AOP annotations (such as @Transactional) on concrete classes 
> instead of only on interfaces. 
> Interface-only AOP annotations will be ignored when CGLIB is enabled. 
> Changing when @Transactional aspects are triggered could lead to items not being saved to the database, 
> or poor performance due to transactional boundary shifting.


# Code Samples
- 后续尽量会在每个知识点里, 都增加对应的完整可运行的code sample, 便于各位学习研究.
- 随着该问题的深入研究, 搞明白了asm, cglib, jdk dynamic proxy, aspectj等的关系, 愈发觉得还是得多搜索英文资料.
- [本篇博文里完整的spring aop code sample](https://github.com/DavyJones2010/head-first-spring/tree/feature/20220603_annotation_on_interface)

# Refs
- [Aspect-Oriented Programming in Spring Boot Part 2: Spring JDK Proxies vs CGLIB vs AspectJ](https://www.credera.com/insights/aspect-oriented-programming-in-spring-boot-part-2-spring-jdk-proxies-vs-cglib-vs-aspectj)
