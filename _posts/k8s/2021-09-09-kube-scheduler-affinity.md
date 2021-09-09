---
layout: post
title: kube-scheduler笔记之亲和性调度
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [k8s, kube-scheduler, scheduler]
lang: zh
---

## 综述

https://github.com/DavyJones2010/k8s-source-code-analysis/blob/master/core/scheduler/affinity.md

## 分类

分为如下几种: 

* NodeSelector

* NodeAffinity
  * preferredDuringSchedulingIgnoredDuringExecution: 软约束
  * requiredDuringSchedulingIgnoredDuringExecution: 硬约束
  
* PodAffinity:
  * preferredDuringSchedulingIgnoredDuringExecution: 软约束
  * requiredDuringSchedulingIgnoredDuringExecution: 硬约束
  
* requiredDuringSchedulingIgnoredDuringExecution

  * 与 NodeSelector 或者 LabelSelector 有啥区别? 都是硬约束, 都会导致Node被筛除掉

    * 区别在于: LabelSelector 只能判断 "a.label.equals(b.label)", 而NodeSelector支持完整的表达式, 例如

      ```yaml
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: kubernetes.io/e2e-az-name
                  operator: In
                  values:
                  - e2e-az1
                  - e2e-az2
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              preference:
                matchExpressions:
                - key: another-node-label-key
                  operator: In
                  values:
                  - another-node-label-value
      ```

* “IgnoredDuringExecution”部分意味着，类似于 `nodeSelector` 的工作原理，如果节点的标签在运行时发生变更，从而不再满足 pod 上的亲和规则，那么 pod 将仍然继续在该节点上运行。也就是onetime的原则, 只会在调度时关注该标签, 后续节点标签变化, 不会重调度. 



## 实现研究

### NodeAffinity

代码样例

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
```



#### RequiredDuringSchedulingIgnoredDuringExecution(硬约束)

在Filter阶段生效

```go
pkg/scheduler/algorithm/predicates/predicates.go:848 podMatchesNodeSelectorAndAffinityTerms
func podMatchesNodeSelectorAndAffinityTerms(pod *v1.Pod, node *v1.Node) bool {
	// NodeSelector, 筛选Node
  selector := labels.SelectorFromSet(pod.Spec.NodeSelector)
  if !selector.Matches(labels.Set(node.Labels)) {
    return false
  }
  
  // NodeAffinity.requiredDuringSchedulingIgnoredDuringExecution, 硬约束
  if nodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution != nil {
			nodeSelectorTerms := nodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution.NodeSelectorTerms
			nodeAffinityMatches = nodeAffinityMatches && nodeMatchesNodeSelectorTerms(node, nodeSelectorTerms)
		}
}


```

#### PreferredDuringSchedulingIgnoredDuringExecution(软约束)

在Weighter阶段生效

```go
pkg/scheduler/algorithm/priorities/node_affinity.go:34 CalculateNodeAffinityPriorityMap

func CalculateNodeAffinityPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulernodeinfo.NodeInfo) (schedulerapi.HostPriority, error) {
   node := nodeInfo.Node()
   affinity := pod.Spec.Affinity

   var count int32

  // 遍历所有Soft约束
  for i := range affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution {
    preferredSchedulingTerm := &affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution[i]

    // 根据表达式生成Selector
    nodeSelector, err := v1helper.NodeSelectorRequirementsAsSelector(preferredSchedulingTerm.Preference.MatchExpressions)
    // 根据Selector计算权重.
    if nodeSelector.Matches(labels.Set(node.Labels)) {
      count += preferredSchedulingTerm.Weight
    }
  }

   // 返回计算好的权重
   return schedulerapi.HostPriority{
      Host:  node.Name,
      Score: int(count),
   }, nil
}
```



### PodAffinity

yaml样例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
  labels:
    app: pod-affinity-pod
spec:
  containers:
  - name: with-pod-affinity
    image: nginx
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - busybox-pod
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - node-affinity-pod
          topologyKey: kubernetes.io/hostname
```

#### RequiredDuringSchedulingIgnoredDuringExecution(硬约束)

* 在filter阶段生效

如何实现的podAntiAffinity? 能强保障么? 


### Service亲和性

*一个服务的第一个Pod被调度到带有Label “region=foo”的Nodes（资源集群）上， 那么其服务后面的其它Pod都将调度至Label “region=foo”的Nodes。*


# 参考
https://www.qikqiak.com/post/understand-kubernetes-affinity/