---
layout: post
title: Java中Log4j的最佳实践
tags: [java, log4j, logging, best-practice]
---

## 通用
- 把ThreadName, RequestId放到MDC, 加到pattern里.

## 异步日志
使用 [log4j2 asyncLogger](https://logging.apache.org/log4j/2.x/manual/async.html#Location) 来异步打印
- 不要在pattern里增加location相关信息, 例如class, file, location, line, method等, 因为这些底层需要[打印堆栈](https://logging.apache.org/log4j/2.x/manual/async.html#Location), 然后遍历, 性能较差. 尤其是需要使用asyncLogger的场景下
- [异步日志底层实现原理](https://issues.apache.org/jira/browse/LOG4J2-2031): 本质是一个RingBuffer, 默认size大约是21w条, 需要注意不要使用DiscardPolicy;  
