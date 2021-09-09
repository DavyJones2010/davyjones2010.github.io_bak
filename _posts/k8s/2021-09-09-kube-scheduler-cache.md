---
layout: post
title: kube-scheduler笔记之SchedulerCache研究
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [k8s, kube-scheduler, scheduler, kube-scheduler-cache]
lang: zh
---

# 源码路径

```
kubernetes/pkg/scheduler/internal/cache/cache.go
```

# 结构详解

## SchedulerCache

### 结构代码

```
type schedulerCache struct {

	// 锁信息
	// This mutex guards all fields within this cache struct.
	mu sync.RWMutex
	
	// Pod信息(重要)
	// a set of assumed pod keys.
	// The key could further be used to get an entry in podStates.
	assumedPods map[string]bool
	// a map from pod key to podState.
	podStates map[string]*podState
	
	// Node信息(重要)
	nodes     map[string]*nodeInfoListItem
	// headNode points to the most recently updated NodeInfo in "nodes". It is the
	// head of the linked list.
	headNode *nodeInfoListItem
	nodeTree *NodeTree
}
```

### 概述

是调度决策的资源信息核心. 包括如下重要部分:

* headNode *nodeInfoListItem: 指向nodeInfo链表的头部指针
* nodes map[string]*nodeInfoListItem: nodeInfo的组合, key: nodeName, value: nodeInfo
* nodeTree *NodeTree: 按zone打散的node结构.

### 核心操作

#### AddNode

```
func (cache *schedulerCache) AddNode(node *v1.Node) error {
	cache.mu.Lock()
	defer cache.mu.Unlock()

	n, ok := cache.nodes[node.Name]
	if !ok {
		n = newNodeInfoListItem(schedulernodeinfo.NewNodeInfo())
		cache.nodes[node.Name] = n
	} else {
		cache.removeNodeImageStates(n.info.Node())
	}
	cache.moveNodeInfoToHead(node.Name)

	cache.nodeTree.AddNode(node)
	cache.addNodeImageStates(node, n.info)
	return n.info.SetNode(node)
}
```

1. 把<nodeName, nodeInfo>放到nodes这个map中.
2. 把当前node append到nodeInfoList链表的头部, 将头部节点指针改成当前node
3. 

#### RemoveNode

与AddNode操作相反.

#### UpdateNodeInfoSnapshot

创建NodeInfo的快照, 每次调度前都会调用该方法, 此后调度都依赖该Snapshot进行决策. 具体参见另外一篇<快照机制>



## NodeTree

### 概述

* 本质上NodeTree是为了在调度时, 让Pod能在不同的zone内不同的Node上能够尽量打散.
* 但如下分析, 在进入调度Filter之前就把Node构建成了树, 然后在Filter的时候, 逐个从NodeTree.next()来pop node信息. 有如下问题: 
  * 业务上: 
    * 那么Pod放在哪个Node上与请求顺序有很大关联. 很可能Filter掉之后, 所有Pod还是聚合在同一个zone下, 甚至同一个Node上. 为啥不在Filter之后将剩余的Node组成NodeTree呢? 
    * Weighter如何进行打分排序? weighter机制是否已经失效? 
  * 性能上: 
    * 所有Node都被缓存起来, 内存开销? 
    * NodeTree是如何保证同步的? 

### 结构代码

```
type NodeTree struct {
    tree      map[string]*nodeArray // 存储zone和zone下面的node信息
    zones     []string              // 存储zones
    zoneIndex int
    numNodes  int
    mu        sync.RWMutex
}
```

```
type nodeArray struct {
    nodes     []string
    lastIndex int
}
```

### 结构分析

* tree map[string] *nodeArray: 
    * 负责存储一个zone下面的所有node节点
    * key: zoneName
    * value: zone下所有的node节点
    * nodeArray:
        * nodes: 该zone下所有node的
        * lastIndex: 记录当前zone分配到的节点索引
* zones: 将调度域下所有的zone名称打平放在zones里
* zoneIndex: 当前分配到的zone的索引

### 如何构建NodeTree?

* 添加node: 
    * 根据node对应的zone(如果zone不存在, 则新增加zone), 从map中获取到nodeArray
    * 将node添加到nodeArray的队列结尾
* 删除node: 
    * 根据node对应的zone, 从map中获取到nodeArray
    * 将node删除后, 将队列里node之后其他节点往前移动一位.
    * 如果node删除后zone为空了, 那么同时将该<zoneName, nodeArray>的kv从map中删除掉. 

### 何时构建NodeTree?

* 何时构建tree?
    * 调度开始, 构建本地cache的时候, 就同时把空的nodeTree构建出来了
* 何时添加node?
    * 
* 何时删除node?
    * 
* 调度的时候, 应该是把所有node打平了, 那么是在什么时候使用这个nodeTree呢? 

* 分配node的流程怎样?
    * zoneIndex与nodeArray同时增加, 

### 分配Node打散算法样例

![image.png](https://imgedu.lagou.com/665f5d51ad30466ba0e9c365d4a7e8cb.jpg)


* 如下结构: 

```
{ZoneA:[NodeA1, NodeA2, NodeA3], ZoneB:[NodeB1, NodeB2], ZoneC:[NodeC1, NodeC2, NodeC3, NodeC4]}
```


* 依次next(), 分配顺序: 

```
ZoneA.NodeA1,ZoneB.NodeB1,ZoneC.NodeC1,
ZoneA.NodeA2,ZoneB.NodeB2,ZoneC.NodeC2,
ZoneA.NodeA3,ZoneC.NodeC3,
ZoneC.NodeC4
-> ResetExhausted(Zone的Index与Zone内Node的Index全部都被重置)
ZoneA.NodeA1,ZoneB.NodeB1,ZoneC.NodeC1,
ZoneA.NodeA2,ZoneB.NodeB2,ZoneC.NodeC2,
ZoneA.NodeA3,ZoneC.NodeC3,
ZoneC.NodeC4
```



## NodeInfoListItem

### 结构代码

```
// nodeInfoListItem holds a NodeInfo pointer and acts as an item in a doubly
// linked list. When a NodeInfo is updated, it goes to the head of the list.
// The items closer to the head are the most recently updated items.
type nodeInfoListItem struct {
	info *schedulernodeinfo.NodeInfo
	next *nodeInfoListItem
	prev *nodeInfoListItem
}
```

### 结构概述

本质上: 是一个NodeInfo的双向链表. (注意头部节点没有prev, 尾部节点没有next, 所以并非环). 一旦NodeInfo更新/添加, 那么该NodeInfo便被放到链表的头部. 如下结构: 

```
+-------+           +--------+        +--------+
|  NodeA| +-------> |   NodeB| +----> |   NodeC|
|       |           |        |        |        |
+-+-----+ <-------+ +--------+ <----+ +--------+
  ^
  |
 Head
```

目的是: 

如何做到: 

* 怎样保证更新之后必然放到链表头部: 参加AddNode操作解析
* 谁会触发NodeInfo的更新? 
* 何时出发NodeInfo的更新?

版本号机制: 

* cache维护了一个全局的版本号(generation).
* NodeInfo结构中, 每个NodeInfo都有一个generation, 每次对链表/Node本身进行操作, 都会把NodeInfo放到链表头. 同时NodeInfo.generation = 全局的版本号+1. 
* 这样就保证了链表头的generation肯定是最大的. 并且链表中从头到尾, generation是从高到低依次递减的.

### 核心操作

#### AddNode

纯链表操作, 比较容易理解. 

* 将当前Node放到NodeInfo链表的头部
* 将头部指针指向当前Node
* 将当前Node的generation+1(推测应该是 当前node的generation=之前头部node的generation + 1), 这样才能保证Node链表的generation是从高到低进行的排列.

```
// moveNodeInfoToHead moves a NodeInfo to the head of "cache.nodes" doubly
// linked list. The head is the most recently updated NodeInfo.
// We assume cache lock is already acquired.
func (cache *schedulerCache) moveNodeInfoToHead(name string) {
	ni, ok := cache.nodes[name]
	if !ok {
		klog.Errorf("No NodeInfo with name %v found in the cache", name)
		return
	}
	// if the node info list item is already at the head, we are done.
	if ni == cache.headNode {
		return
	}

	if ni.prev != nil {
		ni.prev.next = ni.next
	}
	if ni.next != nil {
		ni.next.prev = ni.prev
	}
	if cache.headNode != nil {
		cache.headNode.prev = ni
	}
	ni.next = cache.headNode
	ni.prev = nil
	cache.headNode = ni
}
```

#### RemoveNode

是AddNode的反向操作. 纯链表操作. 不再赘述. 

* 注意这里cache.nodes的map很有用, 能根据Node的Name直接找到Node对象. 复杂度O(1)
* 对比下RemovePod, 需要从头到尾遍历, 复杂度O(n)较高.



### NodeInfo

#### 结构概述

代表单个Node资源信息的最核心部分.



#### 结构代码

```
// NodeInfo is node level aggregated information.
type NodeInfo struct {
	// Overall node information.
	node *v1.Node

	pods             []*v1.Pod
	podsWithAffinity []*v1.Pod
	usedPorts        HostPortInfo

	// Total requested resource of all pods on this node.
	// It includes assumed pods which scheduler sends binding to apiserver but
	// didn't get it as scheduled yet.
	requestedResource *Resource
	nonzeroRequest    *Resource
	// We store allocatedResources (which is Node.Status.Allocatable.*) explicitly
	// as int64, to avoid conversions and accessing map.
	allocatableResource *Resource

	// Cached taints of the node for faster lookup.
	taints    []v1.Taint
	taintsErr error

	// imageStates holds the entry of an image if and only if this image is on the node. The entry can be used for
	// checking an image's existence and advanced usage (e.g., image locality scheduling policy) based on the image
	// state information.
	imageStates map[string]*ImageStateSummary

	// TransientInfo holds the information pertaining to a scheduling cycle. This will be destructed at the end of
	// scheduling cycle.
	// TODO: @ravig. Remove this once we have a clear approach for message passing across predicates and priorities.
	TransientInfo *TransientSchedulerInfo

	// Cached conditions of node for faster lookup.
	memoryPressureCondition v1.ConditionStatus
	diskPressureCondition   v1.ConditionStatus
	pidPressureCondition    v1.ConditionStatus

	// Whenever NodeInfo changes, generation is bumped.
	// This is used to avoid cloning it if the object didn't change.
	generation int64
}
```

#### 结构分析

* requestedResource *Resource: 
  * 已经调度在该Node上的资源总和, 包括两种状态的Pod(参照Pod状态图)
    * Assumed, 但未Binding. 即这些Pod都是在waitForBindingQueue里.
    * Added, 即已经BindToNode
  * 所以理论上存在这种问题, 即单个Node上多个Pod一直都waitForBinding, 导致该Node上资源一直被占用无法被释放. (是否有超时机制?)
  * 如果Pod已经是Binded/Added状态, 那么会从requestResource里删除掉么? 不会.
* allocatableResource *Resource:
  * 该Node上总资源. 应该是totalAmount的概念.
* pods []*v1.Pod: 调度/分配到该Node的Pod队列集合.
* podsWithAffinity []*v1.Pod: TODO: 待研究
* generation: 标识当前Node资源数据的版本号. 每次AddPod, RemovePod操作之后, 都会把版本号+1

* 疑问:
  * 该Node上总资源是多少(totalAmount)? allocatableResource么? 



#### 核心操作

##### AddPod

```
// AddPod adds pod information to this NodeInfo.
func (n *NodeInfo) AddPod(pod *v1.Pod) {
	res, non0CPU, non0Mem := calculateResource(pod)
	n.requestedResource.MilliCPU += res.MilliCPU
	n.requestedResource.Memory += res.Memory
	n.requestedResource.EphemeralStorage += res.EphemeralStorage
	if n.requestedResource.ScalarResources == nil && len(res.ScalarResources) > 0 {
		n.requestedResource.ScalarResources = map[v1.ResourceName]int64{}
	}
	for rName, rQuant := range res.ScalarResources {
		n.requestedResource.ScalarResources[rName] += rQuant
	}
	n.nonzeroRequest.MilliCPU += non0CPU
	n.nonzeroRequest.Memory += non0Mem
	n.pods = append(n.pods, pod)
	if hasPodAffinityConstraints(pod) {
		n.podsWithAffinity = append(n.podsWithAffinity, pod)
	}

	// Consume ports when pods added.
	n.UpdateUsedPorts(pod, true)

	n.generation = nextGeneration()
}
```

###### 流程说明

1. 将Pod请求的资源(cpu, mem等)叠加到Node的requestedResource里.
2. NonzeroRequest代表啥意思? 目前没看明白
3. 将Pod放到Pod队列尾部.
4. 将NodeInfo的版本号+1

##### RemovePod

与AddPod操作完全相反. 不再赘述. 将NodeInfo的版本号+1. 

* 这里吐槽下, remove的时候, 竟然是将Pods的列表, 从头到尾遍历, 当ID等于当前PodID的时候, 执行删除操作. 

#### 疑问总结

1. 在什么时候执行AddPod操作? 
   1. 看起来是在SchedulerCache.addPod, SchedulerCache.updatePod, SchedulerCache.assumePod的时候使用.
   2. 但在什么时候执行呢? 没找到入口. 后续继续看下.
2.

# Snapshot机制

## 重点

1. 打平节点, 深拷贝机制
2. 版本号对比, 增量更新机制

## 代码结构

### NodeInfoSnapshot

```
// NodeInfoSnapshot is a snapshot of cache NodeInfo. The scheduler takes a
// snapshot at the beginning of each scheduling cycle and uses it for its
// operations in that cycle.
type NodeInfoSnapshot struct {
	NodeInfoMap map[string]*schedulernodeinfo.NodeInfo
	Generation  int64
}
```

* NodeInfoMap: <nodeName, NodeInfo>,
    * 即为NodeName到NodeInfo的一个Map, 是从NodeInfo的链表中将所有Node都深拷贝一份, 打平了放在NodeInfoMap里.
    * 同时增加了版本号对比机制, 这样能保证每次打snapshot时, 都是增量深拷贝/更新, 而不是全量.
* Generation: 当前Snapshot对应的最新版本号, 等于打snapshot时, 所有NodeInfo里的最大generation值(即链表头节点的generation).
*



## 重要操作

scheduler_cache中的一个方法.

```
// UpdateNodeInfoSnapshot takes a snapshot of cached NodeInfo map. This is called at
// beginning of every scheduling cycle.
// This function tracks generation number of NodeInfo and updates only the
// entries of an existing snapshot that have changed after the snapshot was taken.
func (cache *schedulerCache) UpdateNodeInfoSnapshot(nodeSnapshot *NodeInfoSnapshot) error {
	cache.mu.Lock()
	defer cache.mu.Unlock()
	balancedVolumesEnabled := utilfeature.DefaultFeatureGate.Enabled(features.BalanceAttachedNodeVolumes)

	// Get the last generation of the the snapshot.
	snapshotGeneration := nodeSnapshot.Generation

	// Start from the head of the NodeInfo doubly linked list and update snapshot
	// of NodeInfos updated after the last snapshot.
	for node := cache.headNode; node != nil; node = node.next {
		if node.info.GetGeneration() <= snapshotGeneration {
			// all the nodes are updated before the existing snapshot. We are done.
			break
		}
		if balancedVolumesEnabled && node.info.TransientInfo != nil {
			// Transient scheduler info is reset here.
			node.info.TransientInfo.ResetTransientSchedulerInfo()
		}
		if np := node.info.Node(); np != nil {
			nodeSnapshot.NodeInfoMap[np.Name] = node.info.Clone()
		}
	}
	// Update the snapshot generation with the latest NodeInfo generation.
	if cache.headNode != nil {
		nodeSnapshot.Generation = cache.headNode.info.GetGeneration()
	}

	if len(nodeSnapshot.NodeInfoMap) > len(cache.nodes) {
		for name := range nodeSnapshot.NodeInfoMap {
			if _, ok := cache.nodes[name]; !ok {
				delete(nodeSnapshot.NodeInfoMap, name)
			}
		}
	}
	return nil
}
```

* 由于Cache中的Node是个双向链表, 且必须保证链表中所有Node的generation是从高到低排序的.
    * 但目前看generation的更新机制, generation不能保证从高到低排序??
