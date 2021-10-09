---
layout: post 
title: 一个简单的Fork&Join框架实现
gh-badge: [star, fork, follow]
tags: [code-snippets, concurrent]
---

# 背景

在代码里, 经常需要类似fork&join的方式来并行完成一些耗时的任务, 并将结果聚合起来. 
通常有如下几种方式来实现:

# 常用实现方式

## countdown latch

## future.get

## java原生的fork&join框架

## CompletionService

## 几种实现方式总结
使用起来都不太优雅: 
1. 需要自己管理线程池
2. 需要自己做结果的聚合
3. 需要自己处理异常情况等
因此我基于CompletionService的方式, 封装了一个小的Fork&Join框架, 来满足一些简单需求.

# 框架参考代码

## 核心部分

```java
package davyjones2010.github.io.concurrent;

import java.util.List;
import java.util.concurrent.CompletionService;
import java.util.concurrent.ExecutorCompletionService;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.LinkedBlockingDeque;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

import com.google.common.collect.Lists;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.concurrent.BasicThreadFactory;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;

/**
 * 一个简单的Fork&Join框架, 用于将批量请求分布到多个线程中, 并且通过completionService队列拿到所有结果
 *
 * @author kunlun.ykl
 */
@Slf4j
//@Service
public class CustomForkJoinService implements InitializingBean, DisposableBean {

    ExecutorService executorService;
    public static final int CORE_SIZE = 32;
    public static final String POOL_NAME = "CFJ";

    public <V> List<CustomForkJoinResult<V>> forkJoin(List<CustomForkJoinCallable<V>> callables) {
        System.out.printf("forkJoin start. request.size: %d \n", callables.size());
        long start = System.currentTimeMillis();

        // hantingtodo: 这里可以增加容灾, 如果executorService初始化失败, 可以降级为串行执行
        CompletionService<CustomForkJoinResult<V>> completionService =
            new ExecutorCompletionService<>(executorService);

        // fork
        for (CustomForkJoinCallable<V> callable : callables) {
            completionService.submit(
                () -> callable.invoke()
            );
        }

        // join
        List<CustomForkJoinResult<V>> results = Lists.newArrayList();
        for (int i = 0; i < callables.size(); i++) {
            try {
                Future<CustomForkJoinResult<V>> take = completionService.take();
                CustomForkJoinResult<V> result = take.get();
                if (null != result && result.success) {
                    results.add(result);
                }
            } catch (Exception e) {
                log.error("invoke error.", e);
            }
        }
        System.out.printf("forkJoin finished. request.size: %d result.size: %d cost: %d \n", callables.size(),
            results.size(),
            System.currentTimeMillis() - start);
        return results;
    }

    @Override
    public void destroy() {
        if (null != executorService) {
            executorService.shutdown();
        }
    }

    @Override
    public void afterPropertiesSet() {
        executorService = new ThreadPoolExecutor(CORE_SIZE, CORE_SIZE, 0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingDeque<>(), new BasicThreadFactory.Builder().namingPattern(POOL_NAME + "-%d").build());
    }
}
```

## Request&Response对象封装

```java
package davyjones2010.github.io.concurrent;

import java.lang.reflect.Method;

import com.alibaba.fastjson.JSON;

import lombok.extern.slf4j.Slf4j;
import org.springframework.util.ReflectionUtils;

@Slf4j
public class CustomForkJoinCallable<V> {
    Object o;
    Object[] params;
    String methodName;
    Class[] paramTypes;

    public CustomForkJoinCallable(Object o, String methodName, Object[] params, Class[] paramTypes) {
        this.o = o;
        this.params = params;
        this.methodName = methodName;
        this.paramTypes = paramTypes;
    }

    public CustomForkJoinResult<V> invoke() {
        CustomForkJoinResult<V> result = new CustomForkJoinResult<>();
        try {
            Method m = ReflectionUtils.findMethod(this.o.getClass(), methodName, paramTypes);
            if (null == m) {
                throw new Exception(
                    "cannot find method: " + methodName + " for class: " + o.getClass().getSimpleName());
            }
            V v = (V)ReflectionUtils.invokeMethod(m, this.o, params);
            result.success = true;
            result.rawResponse = v;
            result.params = params;
        } catch (Exception e) {
            System.out.printf("invoke error. o: %s methodName: %s params: %s \n", o.getClass().getSimpleName(),
                methodName,
                JSON.toJSONString(params));
            e.printStackTrace();
            result.success = false;
        }
        return result;
    }
}
```

```java
package davyjones2010.github.io.concurrent;

public class CustomForkJoinResult<V> {
    public Object[] params;

    public V rawResponse;

    // 代表本次fork&join调用是否成功
    public boolean success;
}
```

## 实现细节说明
之所以封装`CustomForkJoinResult`对象, 是由于我们通常Fork&Join使用时, 不仅要感知实际invoke的结果, 很多时候也需要感知到传入的参数. 
因此将`Object[] params`放入到`CustomForkJoinResult`里, 作为成员变量, 作为**方法调用的上下文**.
例如判断一个String是否有效, 核心的invoke方法签名如下: 
```java
boolean isValid(String a);
```
当Fork&Join框架传入一堆的String时, 如果直接将结果聚合, 由于结果是乱序拿到的,  
那么拿到的是一堆的"boolean"值, 我们根本不知道哪个String对应哪个boolean.
也就不知道哪个String是有效的, 哪个String是无效的了. (具体可以参见下边的测试代码)

# 测试验证

## 测试代码
```java
package davyjones2010.github.io.concurrent;

import java.util.List;

import com.google.common.base.Splitter;
import com.google.common.collect.Lists;
import org.junit.Test;

public class CustomForkJoinServiceTest {
    @Test
    public void forkJoinTest() {
        CustomForkJoinService forkJoinService = new CustomForkJoinService();
        forkJoinService.afterPropertiesSet();
        long start = System.currentTimeMillis();
        List<CustomForkJoinCallable<Boolean>> callables = Lists.newArrayList();
        for (int i = 0; i < 40; i++) {
            Task t = new Task();
            CustomForkJoinCallable<Boolean> callable = new CustomForkJoinCallable<>(t, "isValid",
                new Object[] {"hello-" + i}, new Class[] {String.class});
            callables.add(callable);
        }
        List<CustomForkJoinResult<Boolean>> results = forkJoinService.forkJoin(callables);
        System.out.printf("forkJoin finished. callables.size: %d results.size: %d cost: %d \n", callables.size(),
            results.size(), System.currentTimeMillis() - start);
        for (CustomForkJoinResult<Boolean> result : results) {
            System.out.printf("%s isValid: %b \n", result.params[0], result.rawResponse);
        }
    }

    public static class Task {
        public Boolean isValid(String str) {
            long start = System.currentTimeMillis();
            // 这里模拟耗时操作
            try {
                long l = (long)(Math.random() * 1000);
                Thread.sleep(l);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.printf("echo finished. cost: %d \n", System.currentTimeMillis() - start);
            if (Integer.parseInt(Splitter.on('-').splitToList(str).get(1)) % 7 == 0) {
                return true;
            }
            return false;
        }
    }
}

```
## 测试结果分析

```
forkJoin start. request.size: 40 
echo finished. cost: 29 
echo finished. cost: 35 
echo finished. cost: 124  
...(为了简便省略掉了)
echo finished. cost: 629 
echo finished. cost: 680 
echo finished. cost: 893 
echo finished. cost: 961 
echo finished. cost: 968 
forkJoin finished. request.size: 40 result.size: 40 cost: 1231 
forkJoin finished. callables.size: 40 results.size: 40 cost: 1238 
```
可以看到, 平均单次执行的期望是500ms左右, 如果串行, 耗时应该在 500ms * 40 =20s 左右. 
但使用Fork&Join框架, 最终耗时是1s左右

# 使用限制&改进点
* 木桶效应:
  * 如果某个任务分片执行很久, 例如耗时如: 1, 2, 1, 2, 100, 那么最终的Fork&Join耗时肯定是100ms+;
  * 优化点: 可以增加整体的超时时间, 丢弃掉超时的任务. 以期待部分的返回.
* 线程池: 
  * 这里线程池只是简单地使用了默认的线程池
  * 可以使用带监控能力的线程池, 增加例如活跃线程数, 任务队列堆积情况等
不过整体来说, 基本够用了, 可以无损替换掉之前使用countdownlatch等不太优雅的做法.