---
layout: post
title: SpringBoot Transactional AOP 实现原理研究 
tags: [java, spring, spring-boot, aop, interface, spring-tx]
---

# 背景
书接上回 [SpringBoot基于Interface的Annotation在AOP中如何生效I](https://davyjones2010.github.io/2022-06-03-spring-annotation-on-interface/),
里边只是简略地说明了Spring中 @Transactional 只能在jdk proxy场景下生效, 在cglib场景下不生效, 但没有研究具体代码实现.
本文详细探究下.

# 代码分析
## 0. Spring AOP整体流程
Spring把所有Bean初始化完成, 容器启动前, 会遍历容器中的每个bean, 执行方法, 尝试找到可用的advisor, 生成新的增强bean, 方法入口: `org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findEligibleAdvisors`
详细步骤: 
1. 从Spring Context里找到所有Advisors: `org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findCandidateAdvisors`
2. 根据Advisors里的PointCuts/JoinPoints定义, 判断该bean是否可以被该Advisor增强. `org.springframework.aop.support.AopUtils#canApply(org.springframework.aop.Advisor, java.lang.Class<?>, boolean)`
3. 获取到该bean可以被增强的advisors列表
4. 根据配置`spring.aop.proxy-target-class=true/false`, 使用Cglib或者JDK生成代理bean. `org.springframework.aop.framework.DefaultAopProxyFactory#createAopProxy`
5. 之后从Spring容器中获取bean(getBean或者通过@Autowired等自动注入), 实际上获取到的是代理bean.


## 1. @Transactional 的JoinPoint/PointCut定义

- JoinPoint or PointCut, 在AOP里即是类似SQL中where的条件, 即哪些符合条件的class/bean/method需要被增强.
- JoinPoint标准接口定义如下: 核心是`matches`方法.

```java
public abstract class StaticMethodMatcher implements MethodMatcher {

	@Override
	public final boolean matches(Method method, Class<?> targetClass, Object... args) {
		// should never be invoked because isRuntime() returns false
		throw new UnsupportedOperationException("Illegal MethodMatcher usage");
	}
}
```

- `@Transactional` 使用的是: `org.springframework.transaction.interceptor.TransactionAttributeSourcePointcut`, 而<mark>该pointcut, 会按照类继承关系向上寻找, 找到接口层面定义的annotation.</mark> 
- 详细测试代码参见: [edu.xmu.kunlun.headfirst.spring.tx.aop.BeanFactoryTransactionAttributeSourceAdvisorTest#matchesTest](https://github.com/DavyJones2010/head-first-spring/blob/feature/20220603_annotation_on_interface/src/test/java/edu/xmu/kunlun/headfirst/spring/tx/aop/BeanFactoryTransactionAttributeSourceAdvisorTest.java) 

## 2. @Transactional 的Advice定义

- Advice, 在AOP里即是类似SQL中聚合函数, 即需要对符合条件的class/bean/method做什么处理.
- Advice标准接口详细定义如下: 

```java
public interface MethodInterceptor extends Interceptor {
	Object invoke(@Nonnull MethodInvocation invocation) throws Throwable;
}
```

- `@Transactional` 使用的是: `org.springframework.transaction.interceptor.TransactionInterceptor`, 即会在`invocation.proceed();`前后分别开启与提交事务.

## 3. @Transactional 的Advisor定义

- Advisor or Aspect, 就是 JoinPoint + Advice, 同时需要作为bean注册在Spring中, 方便Spring容器启动时, 执行`从Spring Context里找到所有Advisors`步骤.
- `@Transactional` 使用的是: `org.springframework.transaction.interceptor.BeanFactoryTransactionAttributeSourceAdvisor`
- 所以我们一般都会在 Aspect 上打上 @Aspect + @Component 的标签

## 完整样例

- 如下样例: 

```java
@Aspect // 向Spring容器说明是个Aspect/Advisor
@Component // 向Spring容器注册, 方便找到该bean
public class PerfAspect {
    @Around("@annotation(Perf)") // 作为PointCut定义
    public Object perf(ProceedingJoinPoint joinPoint) throws Throwable { //方法体作为Advice
        long start = System.currentTimeMillis();
        String methodName = joinPoint.getSignature().getName();
        System.out.println(methodName + " start");
        Object o = null;
        try {
            o = joinPoint.proceed();
        } finally {
            System.out.println(methodName + " finished cost: " + (System.currentTimeMillis() - start));
        }
        return o;
    }
}
```

# 实际验证

## 强制使用cglib, 接口上的 @Transactional 能否被正常增强?

- 在springboot配置里, `spring.aop.proxy-target-class=true`
- [TransApiTest#updateTest代码样例](https://github.com/DavyJones2010/head-first-spring/blob/feature/20220603_annotation_on_interface/src/test/java/edu/xmu/kunlun/headfirst/spring/tx/TransApiTest.java)
- 如下发现`TransApi`是被cglib正常增强, 类名: `class edu.xmu.kunlun.headfirst.spring.tx.TransApiImpl$$EnhancerBySpringCGLIB$$2586b9e2`, 事务能被正常启动

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206062333446.png)



## 强制使用jdk proxy, 接口上的 @Transactional 能否被正常增强?

- 在springboot配置里, `spring.aop.proxy-target-class=false`
- [TransApiTest#updateTest代码样例](https://github.com/DavyJones2010/head-first-spring/blob/feature/20220603_annotation_on_interface/src/test/java/edu/xmu/kunlun/headfirst/spring/tx/TransApiTest.java)
- 如下发现`TransApi`是被jdk正常增强, 类名: `class com.sun.proxy.$Proxy56`, 事务能被正常启动

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206062335278.png)

- httodo: 这里与spring官方文档说明有所不同. 根据上述代码分析, 其实到底能不能增强, 与到底使用cglib还是使用jdk-proxy没啥关系, 只与JoinPoint/PointCut的实现有关系. 待最终探究确认.

# Code Samples
- [本篇博文里完整的spring aop code sample](https://github.com/DavyJones2010/head-first-spring/tree/feature/20220603_annotation_on_interface)

# Refs
