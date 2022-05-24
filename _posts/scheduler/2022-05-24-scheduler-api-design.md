---
layout: post
title: 资源调度API设计的思考
tags: [scheduler-design, api-design, scheduler-api]
lang: zh
---


# 前言
记录下调度API实际设计与应用场景中, 一些有意义的思考与沉淀.

# 批量调度

## amount & minAmount 模式: 

### 模式解释

样例请求如下:

```java
class BatchParam {
    Integer minAmount;
    Integer amount;    
}
```

语义解释: 
- minAmount: 代表请求方需求的最小的资源量. 
- amount: 代表请求方需要的最大资源量. 

有点儿抽象, 我们站在scheduler的角度看. 
批量调度资源, 资源不一定总会满足用户需求的资源, 此时就需要做个妥协,  
如果能调度出来的资源量(realAmount)范围:
- [0, minAmount), 则代表无法满足用户最小需求量, 直接返回失败, 不扣减任何资源.
- [minAmount, amount), 则代表满足用户最小需求量, 则返回部分成功, 扣减realAmount的资源量, 并把realAmount透出给调用方.
- ==amount, 则代表可以完全满足用户资源需求, 则返回成功, 扣减realAmount=amount的资源量.

### 模式样例
这种双参数的方案, 本质是单参数 `amount` 的超集. 更加灵活.

- allOrNothing模式: 严格要求必须是amount, 则参数中 minAmount = amount 即可实现.
- tryBest模式: 能创建几个是几个, 但最好能创建出amount数量. 则参数中, minAmount = 0 即可实现.

而实际应用上, 阿里云的API [RunInstances](https://help.aliyun.com/document_detail/63440.html) 也是如此设计的.


## tryBest & allOrNothing 模式:
本质上可以用上边的 `minAmount, amount` 两个参数进行描述, 更加优雅与灵活.


## 批量调度性能优化的思考






