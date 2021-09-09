---
layout: post
title: kube-scheduler笔记之filter
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [k8s, kube-scheduler, scheduler]
lang: zh
---

## 概述

* Predicates Policies就是提供给Scheduler用来过滤出满足所定义条件的Nodes，**并发的**(最多16个goroutine)对每个Node启动所有Predicates Policies的遍历Filter，看其是否都满足配置的Predicates Policies，若有一个Policy不满足，则直接被淘汰。



## 核心代码

```go
// Filters the nodes to find the ones that fit based on the given predicate functions
// Each node is passed through the predicate functions to determine if it is a fit
func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error) {
	var filtered []*v1.Node
	failedPredicateMap := FailedPredicateMap{}

	if len(g.predicates) == 0 {
		filtered = nodes
	} else {
		allNodes := int32(g.cache.NodeTree().NumNodes())
		numNodesToFind := g.numFeasibleNodesToFind(allNodes)

		// Create filtered list with enough space to avoid growing it
		// and allow assigning.
		filtered = make([]*v1.Node, numNodesToFind)
		errs := errors.MessageCountMap{}
		var (
			predicateResultLock sync.Mutex
			filteredLen         int32
		)

		ctx, cancel := context.WithCancel(context.Background())

		// We can use the same metadata producer for all nodes.
		meta := g.predicateMetaProducer(pod, g.nodeInfoSnapshot.NodeInfoMap)

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
			if err != nil {
				predicateResultLock.Lock()
				errs[err.Error()]++
				predicateResultLock.Unlock()
				return
			}
			if fits {
				length := atomic.AddInt32(&filteredLen, 1)
				if length > numNodesToFind {
					cancel()
					atomic.AddInt32(&filteredLen, -1)
				} else {
					filtered[length-1] = g.nodeInfoSnapshot.NodeInfoMap[nodeName].Node()
				}
			} else {
				predicateResultLock.Lock()
				failedPredicateMap[nodeName] = failedPredicates
				predicateResultLock.Unlock()
			}
		}

		// Stops searching for more nodes once the configured number of feasible nodes
		// are found.
		workqueue.ParallelizeUntil(ctx, 16, int(allNodes), checkNode)

		filtered = filtered[:filteredLen]
		if len(errs) > 0 {
			return []*v1.Node{}, FailedPredicateMap{}, errors.CreateAggregateFromMessageCountMap(errs)
		}
	}

	if len(filtered) > 0 && len(g.extenders) != 0 {
		for _, extender := range g.extenders {
			if !extender.IsInterested(pod) {
				continue
			}
			filteredList, failedMap, err := extender.Filter(pod, filtered, g.nodeInfoSnapshot.NodeInfoMap)
			if err != nil {
				if extender.IsIgnorable() {
					klog.Warningf("Skipping extender %v as it returned error %v and has ignorable flag set",
						extender, err)
					continue
				} else {
					return []*v1.Node{}, FailedPredicateMap{}, err
				}
			}

			for failedNodeName, failedMsg := range failedMap {
				if _, found := failedPredicateMap[failedNodeName]; !found {
					failedPredicateMap[failedNodeName] = []predicates.PredicateFailureReason{}
				}
				failedPredicateMap[failedNodeName] = append(failedPredicateMap[failedNodeName], predicates.NewFailureReason(failedMsg))
			}
			filtered = filteredList
			if len(filtered) == 0 {
				break
			}
		}
	}
	return filtered, failedPredicateMap, nil
}
```





### PodFitsOnNode

```go
func podFitsOnNode(
	pod *v1.Pod,
	meta predicates.PredicateMetadata,
	info *schedulernodeinfo.NodeInfo,
	predicateFuncs map[string]predicates.FitPredicate,
	queue internalqueue.SchedulingQueue,
	alwaysCheckAllPredicates bool,
) (bool, []predicates.PredicateFailureReason, error) {
	var failedPredicates []predicates.PredicateFailureReason

	podsAdded := false

	for i := 0; i < 2; i++ {
		metaToUse := meta
		nodeInfoToUse := info
		if i == 0 {
			podsAdded, metaToUse, nodeInfoToUse = addNominatedPods(pod, meta, info, queue)
		} else if !podsAdded || len(failedPredicates) != 0 {
			break
		}
		for _, predicateKey := range predicates.Ordering() {
			var (
				fit     bool
				reasons []predicates.PredicateFailureReason
				err     error
			)
			
			if predicate, exist := predicateFuncs[predicateKey]; exist {
				fit, reasons, err = predicate(pod, metaToUse, nodeInfoToUse)
				if err != nil {
					return false, []predicates.PredicateFailureReason{}, err
				}

				if !fit {
					// eCache is available and valid, and predicates result is unfit, record the fail reasons
					failedPredicates = append(failedPredicates, reasons...)
					// if alwaysCheckAllPredicates is false, short circuit all predicates when one predicate fails.
					if !alwaysCheckAllPredicates {
						klog.V(5).Infoln("since alwaysCheckAllPredicates has not been set, the predicate " +
							"evaluation is short circuited and there are chances " +
							"of other predicates failing as well.")
						break
					}
				}
			}
		}
	}

	return len(failedPredicates) == 0, failedPredicates, nil
}
```



## 核心概念

### NominatedPods

当enable PodPriority feature gate后，scheduler会在集群资源资源不足时为preemptor抢占低优先级的Pods（成为victims）的资源，然后preemptor会再次入调度队列，等待下次victims的优雅终止并进行下一次调度。

为了尽量避免从preemptor抢占资源到真正再次执行调度这个时间段的scheduler能感知到那些资源已经被抢占，在scheduler调度其他更低优先级的Pods时考虑这些资源已经被抢占，因此在抢占阶段，为给preemptor设置`pod.Status.NominatedNodeName`，表示在NominatedNodeName上发生了抢占，preemptor期望调度在该node上。

PriorityQueue中缓存了每个node上的NominatedPods，这些NominatedPods表示已经被该node提名的，期望调度在该node上的，但是又还没最终成功调度过来的Pods。

### CriticalPod&NonCriticalPod

https://cloud.tencent.com/developer/article/1402111

CriticalPod, 用来部署关键组件, 希望能够: 

1. 调度时, 即使资源不足, 仍然能够调度上去. 
2. 不能被抢占 



## 核心流程

1. 根据NodeSize, 确定Filter之后能参与后续Weighter的Node个数
2. g.cache.NodeTree().Next(): 
   1. 从cache的NodeTree中依次获取NodeName, 便于Node打散. 
   2. 从snapshot的中根据NodeName获取到Node对象.
3. podFitsOnNode(): 后续调度的
4. 

```Go
var (
	predicatesOrdering = []string{CheckNodeConditionPred, CheckNodeUnschedulablePred,
		GeneralPred, HostNamePred, PodFitsHostPortsPred,
		MatchNodeSelectorPred, PodFitsResourcesPred, NoDiskConflictPred,
		PodToleratesNodeTaintsPred, PodToleratesNodeNoExecuteTaintsPred, CheckNodeLabelPresencePred,
		CheckServiceAffinityPred, MaxEBSVolumeCountPred, MaxGCEPDVolumeCountPred, MaxCSIVolumeCountPred,
		MaxAzureDiskVolumeCountPred, MaxCinderVolumeCountPred, CheckVolumeBindingPred, NoVolumeZoneConflictPred,
		CheckNodeMemoryPressurePred, CheckNodePIDPressurePred, CheckNodeDiskPressurePred, MatchInterPodAffinityPred}
)
```











### 资源量Filter/Predicates

#### NonCriticalPredicates

```go
pkg/scheduler/algorithm/predicates/predicates.go:1114 GeneralPredicates
pkg/scheduler/algorithm/predicates/predicates.go:1136 noncriticalPredicates
pkg/scheduler/algorithm/predicates/predicates.go:769 PodFitsResources

func PodFitsResources(pod *v1.Pod, meta PredicateMetadata, nodeInfo *schedulernodeinfo.NodeInfo) (bool, []PredicateFailureReason, error) {
	node := nodeInfo.Node()

	var predicateFails []PredicateFailureReason
	allowedPodNumber := nodeInfo.AllowedPodNumber()
  // Pod数量校验
	if len(nodeInfo.Pods())+1 > allowedPodNumber {
		predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourcePods, 1, int64(len(nodeInfo.Pods())), int64(allowedPodNumber)))
	}

	var podRequest *schedulernodeinfo.Resource
	podRequest = GetResourceRequest(pod)
	allocatable := nodeInfo.AllocatableResource()
  // CPU量校验
	if allocatable.MilliCPU < podRequest.MilliCPU+nodeInfo.RequestedResource().MilliCPU {
		predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceCPU, podRequest.MilliCPU, nodeInfo.RequestedResource().MilliCPU, allocatable.MilliCPU))
	}
  // MEM量校验
	if allocatable.Memory < podRequest.Memory+nodeInfo.RequestedResource().Memory {
		predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceMemory, podRequest.Memory, nodeInfo.RequestedResource().Memory, allocatable.Memory))
	}
	return len(predicateFails) == 0, predicateFails, nil
}
```

#### CriticalPredicates

```go
pkg/scheduler/algorithm/predicates/predicates.go:1150 EssentialPredicates
func EssentialPredicates(pod *v1.Pod, meta PredicateMetadata, nodeInfo *schedulernodeinfo.NodeInfo) (bool, []PredicateFailureReason, error) {
	var predicateFails []PredicateFailureReason
  // 1. PodFitsHost校验: (pod.Spec.NodeName == node.Name)
	fit, reasons, err := PodFitsHost(pod, meta, nodeInfo)
  // 2. PodFitsHostPorts校验: 端口量是否充足
	fit, reasons, err = PodFitsHostPorts(pod, meta, nodeInfo)
  // 3. PodMatchNodeSelector校验: 
	fit, reasons, err = PodMatchNodeSelector(pod, meta, nodeInfo)

	return len(predicateFails) == 0, predicateFails, nil
}


```



### Label匹配

```go
pkg/scheduler/algorithm/predicates/predicates.go:948

func (n *NodeLabelChecker) CheckNodeLabelPresence(pod *v1.Pod, meta PredicateMetadata, nodeInfo *schedulernodeinfo.NodeInfo) (bool, []PredicateFailureReason, error) {
	node := nodeInfo.Node()
  
	var exists bool
  // 获取Node上的所有Label
	nodeLabels := labels.Set(node.Labels)
  // 遍历pod请求里的所有Label
	for _, label := range n.labels {
    // 如果Node的Label没有包含pod请求里的Label, 那么证明该Node不合法(缺少对应的Label), 则筛除掉
		exists = nodeLabels.Has(label)
		if (exists && !n.presence) || (!exists && n.presence) {
			return false, []PredicateFailureReason{ErrNodeLabelPresenceViolated}, nil
		}
	}
	return true, nil, nil
}


```



## 参考资料

https://my.oschina.net/jxcdwangtao/blog/1818975

https://my.oschina.net/jxcdwangtao/blog/826741





