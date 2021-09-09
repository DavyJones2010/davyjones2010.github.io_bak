---
layout: post
title: k8s基础架构笔记
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [k8s, k8s-architect]
lang: zh
---

## 基础应用组件

* APIServer
  * 认证
  * 鉴权
  * 准入控制
    * Quota
* Scheduler
* Kubelet
* Etcd
* Kube-Controller Manager包括
  * ReplicaSet Controller
  * CDR的Controller
  * AdmissionController 
  * 等


## 资源分为两类
* Cluster-scoped resources
    * 如node
* Namespace-scoped resources
    * 如pod


## 基础流程

![基础流程](https://hugo-picture.oss-cn-beijing.aliyuncs.com/what-happens-when-k8s.svg)



## 基础概念

* workloads



## 基础组件

* Node
* Pod 

## Controller控制组件

* ReplicationController(弃用) --> ReplicaSet(类似ESS)
* Deployment: 后端用ESS, 区别是什么? 
* DaemonSet: Pod组, 部署在每个Node上. 新加入Node, 会自动新部署Pod. 下线Node, 会自动删除Pod.
* StatefulSets: 
* Job&CronJob: 

## 编排组件

* Operator = 资源(CRD) + Controller



## 其他

* CDR
* OAM






