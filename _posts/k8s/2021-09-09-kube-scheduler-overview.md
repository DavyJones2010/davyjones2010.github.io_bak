---
layout: post
title: kube-scheduler笔记
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [k8s, kube-scheduler, scheduler]
lang: zh
---

## 一言概之

将PodSpec.NodeName为空的Pods逐个地，经过预选(Predicates)和优选(Priorities)两个步骤，挑选最合适的Node作为该Pod的Destination。


## 架构概述

1. kube-scheduler作为kubernetes master上一个单独的进程提供调度服务，通过--master指定kube-api-server的地址，用来watch pod和node和调用api server bind接口完成node和pod的Bind操作。
2. kube-scheduler中维护了一个FIFO类型的PodQueue cache，新创建的Pod都会被ConfigFactory watch到，被添加到该PodQueue中，每次调度都从该PodQueue中getNextPod作为即将调度的Pod。
3. 获取到待调度的Pod后，就执行AlgorithmProvider配置Algorithm的Schedule方法进行调度，整个调度过程分两个关键步骤：Predicates和Priorities，最终选出一个最适合该Pod借宿的Node返回。
4. 更新SchedulerCache中Pod的状态(AssumePod)，标志该Pod为scheduled，并更新到对应的NodeInfo中。
5. 调用api server的Bind接口，完成node和pod的Bind操作，如果Bind失败，从SchedulerCache中删除上一步中已经Assumed的Pod。



## 重要疑问

1. 如何实现亲和性/反亲和性调度?
   1. 参见亲和性调度部分
2. 如何做到Weighter?
3. NC实际资源与Scheduler缓存的资源一致性如何保证?
   1. snapshot机制, 无法保证, 只是在调度前都拿该资源
   2. 同时将资源的操作都在scheduler进行收口
4. 如果单个Pod调度失败(无论是在Filter&Weigher阶段, 还是在assume阶段, 还是在Bind阶段), 这个Pod之后还会被重试调度么? 还是说直接失败? 
5. 是否有类似Reservation功能呢? 
6. 抢占式调度是如何实现的? 
   1. 参见抢占式调度部分
7. 调度发生时机是怎样的? 是否支持热迁移? 是否有离线规划? 
   1. 无热迁移需求. 
   2. 只有ASI有离线规划情况.


## 结构分析

```go
type genericScheduler struct {
	cache                    internalcache.Cache
	schedulingQueue          internalqueue.SchedulingQueue
	predicates               map[string]predicates.FitPredicate
	priorityMetaProducer     priorities.PriorityMetadataProducer
	predicateMetaProducer    predicates.PredicateMetadataProducer
	prioritizers             []priorities.PriorityConfig
	pluginSet                pluginsv1alpha1.PluginSet
	extenders                []algorithm.SchedulerExtender
	lastNodeIndex            uint64
	alwaysCheckAllPredicates bool
	nodeInfoSnapshot         internalcache.NodeInfoSnapshot
	volumeBinder             *volumebinder.VolumeBinder
	pvcLister                corelisters.PersistentVolumeClaimLister
	pdbLister                algorithm.PDBLister
	disablePreemption        bool
	percentageOfNodesToScore int32
}
```

* cache: 调度的cache
* nodeInfoSnapshot: node的snapshot, 详细分析参见"Scheduler Cache研究"
* schedulingQueue:  
  * 参见: pkg/scheduler/internal/queue/scheduling_queue.go
  * 本质上是PriorityQueue, 详细分析参见: "Scheduler Queue研究"
* 



## 关键操作

### 1. ScheduleOne

#### 1.1 代码

```go
pkg/scheduler/scheduler.go:436

func (sched *Scheduler) scheduleOne() {
	// 1. 从??获取到要调度的Pod信息
  pod := sched.config.NextPod()
  // 2. 执行调度, 找到合适的Node
	scheduleResult, err := sched.schedule(pod)

	// 3. 执行Node资源扣减
  assumedPod := pod.DeepCopy()
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)

	// 4. 执行Bind流程
	go func() {
		// 4.1 执行prebind, 如果无法bind通过, 则将Node资源归还
		for _, pl := range plugins.PrebindPlugins() {
			approved, err := pl.Prebind(plugins, assumedPod, scheduleResult.SuggestedHost)
			if !approved {
				forgetErr := sched.Cache().ForgetPod(assumedPod)
				return
			}
		}
		// 4.2 执行bind, 如果无法bind通过, 此bind方法里将Node资源归还
		err := sched.bind(assumedPod, &v1.Binding{
			ObjectMeta: metav1.ObjectMeta{Namespace: assumedPod.Namespace, Name: assumedPod.Name, UID: assumedPod.UID},
			Target: v1.ObjectReference{
				Kind: "Node",
				Name: scheduleResult.SuggestedHost,
			},
		})
	}()
}
```

#### 1.2 操作详解

* 本质上就分为上边4步, 清晰易懂.



### 2. Schedule

#### 1.1 代码

```go
generic_scheduler.go

// Schedule tries to schedule the given pod to one of the nodes in the node list.
// If it succeeds, it will return the name of the node.
// If it fails, it will return a FitError error with reasons.
func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (result ScheduleResult, err error) {

// 1. 获取最新Node信息
	nodes, err := nodeLister.List()
// 2. 用Node信息更新到scheduler的缓存中, 打个快照
  g.snapshot();
// 3. 执行filter
	filteredNodes, failedPredicateMap, err := g.findNodesThatFit(pod, nodes)
// 4. 执行weighter
	metaPrioritiesInterface := g.priorityMetaProducer(pod, g.nodeInfoSnapshot.NodeInfoMap)
	priorityList, err := PrioritizeNodes(pod, g.nodeInfoSnapshot.NodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
// 如果相同priority, 则根据roundrobin算法从list中获取随机NC
	host, err := g.selectHost(priorityList)
	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(filteredNodes) + len(failedPredicateMap),
		FeasibleNodes:  len(filteredNodes),
	}, err
}
```

#### 1.2. 操作详解

1. nodeLister.list? 需要整体看下list机制.
2. g.snapshot(): 给所有Node打一次快照. 具体参见Cache Snapshot机制

### 2. findNodesThatFit

#### 2.1 代码

```go
func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error) {
  
	var filtered []*v1.Node

  // 1. 根据所有Node个数, 确定最终Filter之后最大的Node个数, 缩小之后Weighter的范围
	allNodes := int32(g.cache.NodeTree().NumNodes())
	numNodesToFind := g.numFeasibleNodesToFind(allNodes)
	filtered = make([]*v1.Node, numNodesToFind)
	meta := g.predicateMetaProducer(pod, g.nodeInfoSnapshot.NodeInfoMap)

  // 2. 根据NodeTree结构, 找下一个Node, 判断Node资源是否能满足Pod资源请求
	checkNode := func(i int) {
		nodeName := g.cache.NodeTree().Next()
		fits, failedPredicates, err := podFitsOnNode(
			pod,
			meta,
			g.nodeInfoSnapshot.NodeInfoMap[nodeName],
			g.predicates,
			g.schedulingQueue,
			g.alwaysCheckAllPredicates,
		)
		if fits {
			length := atomic.AddInt32(&filteredLen, 1)
			if length > numNodesToFind {
				atomic.AddInt32(&filteredLen, -1)
			} else {
				filtered[length-1] = g.nodeInfoSnapshot.NodeInfoMap[nodeName].Node()
			}
		}
	}

  // 3. 类似Fork&Join, 启动16个协程并行进行Filter
	workqueue.ParallelizeUntil(ctx, 16, int(allNodes), checkNode)

	return filtered, failedPredicateMap, nil
}
```

#### 2.1 操作详解

1. 遍历Node, 使用的是NodeTree.next()机制, 这样就必须保证NodeTree.next()是线程安全的. K8S里采用mutex信号量来保障
2. 具体NodeTree的机制, 参见单独文章.

### 3. PodFitsResources

#### 3.1 核心代码

```go
pkg/scheduler/algorithm/predicates/predicates.go:769

func PodFitsResources(pod *v1.Pod, meta PredicateMetadata, nodeInfo *schedulernodeinfo.NodeInfo) (bool, []PredicateFailureReason, error) {
  node := nodeInfo.Node()

  // 1. 检查是否超过NC上允许调度的Pod个数上限
	allowedPodNumber := nodeInfo.AllowedPodNumber()
	if len(nodeInfo.Pods())+1 > allowedPodNumber {
		predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourcePods, 1, int64(len(nodeInfo.Pods())), int64(allowedPodNumber)))
	}

  // 2. 计算出Pod的资源请求量
	var podRequest *schedulernodeinfo.Resource
	podRequest = GetResourceRequest(pod)

	allocatable := nodeInfo.AllocatableResource()
  // 3. 判断Node.totalVcpu - podRequest.Vcpu - Node.usedVcpu是否OK
	if allocatable.MilliCPU < podRequest.MilliCPU+nodeInfo.RequestedResource().MilliCPU {
		predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceCPU, podRequest.MilliCPU, nodeInfo.RequestedResource().MilliCPU, allocatable.MilliCPU))
	}
	if allocatable.Memory < podRequest.Memory+nodeInfo.RequestedResource().Memory {
		predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceMemory, podRequest.Memory, nodeInfo.RequestedResource().Memory, allocatable.Memory))
	}
	return len(predicateFails) == 0, predicateFails, nil
}
```
#### 3.2 操作详解

1. Node上重要的几个资源字段:
   1. nodeInfo.AllocatableResource(): 代表该Node上所有的CPU量. 与openstack的totalCpu相同
   2. podRequest.MilliCPU: 代表该新Pod所需求的CPU量. 
   3. nodeInfo.RequestedResource(): 
      1. 代表已经调度到该Node上Pods已经占用的资源. 与openstack的usedCpu相同
      2. 包含了已经assumed的, 但还没有实际bind到Node上的Pod资源. 

### 4. assume

#### 4.1 核心代码

```go

func (sched *Scheduler) assume(assumed *v1.Pod, host string) {
	assumed.Spec.NodeName = host
	sched.config.SchedulerCache.AssumePod(assumed);
}

func (cache *schedulerCache) AssumePod(pod *v1.Pod) error {
	key, err := schedulernodeinfo.GetPodKey(pod)

	// podName之前不能在podStates里
	if _, ok := cache.podStates[key]; ok {
		return fmt.Errorf("pod %v is in the cache, so can't be assumed", key)
	}

	cache.addPod(pod)
	ps := &podState{
		pod: pod,
	}
	cache.podStates[key] = ps
	cache.assumedPods[key] = true
	return nil
}

func (cache *schedulerCache) addPod(pod *v1.Pod) {
	n, ok := cache.nodes[pod.Spec.NodeName]	
  // 见下方详解
	n.info.AddPod(pod)
 // 将Node移动到双链表的头部
	cache.moveNodeInfoToHead(pod.Spec.NodeName)
}

func (n *NodeInfo) AddPod(pod *v1.Pod) {
  // 1. 计算Pod所需资源
	res, non0CPU, non0Mem := calculateResource(pod)
  // 2. 将Pod所需资源叠加到Node.requestedResource上
	n.requestedResource.MilliCPU += res.MilliCPU
	n.requestedResource.Memory += res.Memory
  // 3. 将Pod得加到Node.pods列表上
	n.pods = append(n.pods, pod)
  // 4. 更新Node.generation
	n.generation = nextGeneration()
}
```

如下重要操作次序:

* 将pod放到cache中Node的pods[]列表中
* 将pod需求的资源量叠加到Node的requestedResource中(注意在调度filter的时候, 判断已经占用的资源, 用的就是这个字段).
* 更新Node的generation, 将Node放到Node双向链表的头部(方便打增量snapshot)

疑问&关注点:

* 如何防止Node资源超卖? 即将pod放到Node的pods[]列表中的时候, Node本身资源发生了变化, 导致useResource>totalResource?
* 



### 5. bind

#### 5.1 核心代码

Binding结构核心: 

```Go
err := sched.bind(assumedPod, &v1.Binding{
   ObjectMeta: metav1.ObjectMeta{Namespace: assumedPod.Namespace, Name: assumedPod.Name, UID: assumedPod.UID},
   Target: v1.ObjectReference{
      Kind: "Node",
      Name: scheduleResult.SuggestedHost,
   },
})
```

bind逻辑核心:

```Go
func (sched *Scheduler) bind(assumed *v1.Pod, b *v1.Binding) error {
  // 1. 获取Binder, 并调用RPC接口, 让对应Node执行绑定Pod
   err := sched.config.GetBinder(assumed).Bind(b)
   if err != nil {
      sched.config.SchedulerCache.ForgetPod(assumed)
      return err
   }
   return nil
}
```

逆向流程: 

```Go
func (cache *schedulerCache) ForgetPod(pod *v1.Pod) error {
  // 1. 获取PodID
   key, err := schedulernodeinfo.GetPodKey(pod)

   currState, ok := cache.podStates[key]
   case ok && cache.assumedPods[key]:
       // 2. 将pod从SchedulerCache中移除
      err := cache.removePod(pod)
      delete(cache.assumedPods, key)
      delete(cache.podStates, key)
   return nil
}

func (cache *schedulerCache) removePod(pod *v1.Pod) error {
  // 1. 找到该Pod调度到的Node, 从NodeInfo中移除该Pod
	n, ok := cache.nodes[pod.Spec.NodeName]
  err := n.info.RemovePod(pod)
	if len(n.info.Pods()) == 0 && n.info.Node() == nil {
    // 2. 将空的Node从链表中移除
		cache.removeNodeInfoFromList(pod.Spec.NodeName)
	} else {
    // 3. 将Node放到双链表头部
		cache.moveNodeInfoToHead(pod.Spec.NodeName)
	}
	return nil
}

func (n *NodeInfo) RemovePod(pod *v1.Pod) error {
	k1, err := GetPodKey(pod)
	for i := range n.pods {
		k2, err := GetPodKey(n.pods[i])
		if k1 == k2 {
			// 将Pod从Node的Pods列表中移除
			n.pods[i] = n.pods[len(n.pods)-1]
			n.pods = n.pods[:len(n.pods)-1]
			// 归还Node资源
			res, non0CPU, non0Mem := calculateResource(pod)
			n.requestedResource.MilliCPU -= res.MilliCPU
			n.requestedResource.Memory -= res.Memory
      // 更新Node版本号
			n.generation = nextGeneration()
			return nil
		}
	}
	return fmt.Errorf("no corresponding pod %s in pods of node %s", pod.Name, n.node.Name)
}
```



#### 5.2 操作详解

1. Binding对象: 本质上是{Pod.UID, Node.nodeName}, 
2. 执行绑定流程: 本质上是调用RPC, 将该Binding对象发送到对应Node上, 执行绑定
   1. 此处RPC模式是怎样的需要确认:  TODO
      1. Pub-Sub模式: scheduler将请求发送到, Node上agent监听到该请求, 发现自身NodeId与消息中相同, 则执行bind过程. 
      2. P2P模式: scheduler直接将请求发送到对应Node上, 执行. 
   2. 此处RPC是同步还是一部需要确认: TODO
      1. 同步: 当前逆向流程即可覆盖
      2. 异步: 需要单独注册回调, 在哪里? 怎么执行?
   3. 具体到Node上, kubelet执行的操作是怎样的, 同样需要确认. TODO
3. 逆向流程: 本质上是将Pod资源归还给Node.



## 调度请求对象

https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

```Go
type ResourceRequirements struct {
	Limits ResourceList `json:"limits,omitempty" protobuf:"bytes,1,rep,name=limits,casttype=ResourceList,castkey=ResourceName"`
	Requests ResourceList `json:"requests,omitempty" protobuf:"bytes,2,rep,name=requests,casttype=ResourceList,castkey=ResourceName"`
}
```

* limits: 给kubelets用的, 指的是这个Pod能使用的最大资源量
* requests: 给scheduler用的, 指的是这个Pod最小需求的资源量


## 参考资料

https://my.oschina.net/jxcdwangtao/blog/824965







