---
layout: post
title: kube-scheduler笔记之Queue研究
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [k8s, kube-scheduler, scheduler, kube-scheduler-queue]
lang: zh
---


## 代码结构

```Go
type PriorityQueue struct {

   // backoff队列
   podBackoff *util.PodBackoff
   // backoff队列, 与podBackoff区别?
   podBackoffQ *util.Heap
  
   // activeQ is heap structure that scheduler actively looks at to find pods to
   // schedule. Head of heap is the highest priority pod.
   activeQ *util.Heap

   // unschedulable队列
   unschedulableQ *UnschedulablePodsMap
  
   // 抢占式调度成功后, 将preemptor放到nominatedPodMap里
   nominatedPods *nominatedPodMap
  
   // 全局调度周期的递增序号，当pod pop的时候会递增
   schedulingCycle int64
   // 当未调度的pod重新被添加到activeQueue中会保存schedulingCycle到moveRequestCycle中
   moveRequestCycle int64
}
```



## 核心解读

1. Heap结构, 即为最大堆.
2. activeQ: 存储所有等待调度的Pod的队列, 根据pod.Spec.Priority来构建最大堆
3. podBackoffQ: 存储在多个schedulingCycle中依旧调度失败的情况下，则会通过之前说的backOff机制，延迟等待调度的时间. 但该backoffQ是纯的PodHeap, 并没有存储pod的backOff的具体信息
4. podBackoff: 类似一个记分板, backoff的计数器，最后一次更新的时间等
5. unschedulableQ: 本质上是<podName, podInfo>的map, 而不是一个队列.



## 重点疑问

1. 在什么时候Pod被加入到activeQ中? 何时加入podBackoffQ中? 何时加入unschedulableQ中? Pod在这几个Q中间是如何变化的?

    1.

2. podBackoffQ是按照什么来划分优先级的?

    1. 如果仍然是按照优先级来, 如何防止高优先级Pod反复拉起, 反复失败?
    2. 应该是按照调度失败次数? 还是调度等待时长?
    3. 根据podsCompareBackoffCompleted, 按照podBackoffTime, 即调度失败次数. 调度等待时长与调度失败次数有一个函数关系.
        1. 如果pod从podBackoffQ中移入到activeQ中, 接下来调度失败, 重新移入podBackoffQ中, 此时podBackoffTime会重置么?

3. 从unschedulableQ中移出pod的优先级是怎样的?

    1.

4. Pod调度失败的终止条件是什么? 还是说没有终止条件? 始终在几个Queue中打转?

    1. 对的. 除非显式调用api删除掉该Pod

5. 各个队列有大小限制么? 如何防止Pod请求积压过多导致Queue(内存)满了?



![img](https://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltZzIwMTguY25ibG9ncy5jb20vYmxvZy8xNTA2NzI0LzIwMjAwMS8xNTA2NzI0LTIwMjAwMTEzMTEwMzU5Mzg1LTM2MDkzMzYzNS5wbmc=.jpg)



## 参考资料

https://www.shuzhiduo.com/A/rV576rRXJP/
