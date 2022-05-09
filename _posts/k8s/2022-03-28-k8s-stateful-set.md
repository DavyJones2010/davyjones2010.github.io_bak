---
layout: post
title: k8s statefulset的研究与实践
tags: [k8s, stateful-set]
lang: zh
---

# 背景
一直很感兴趣, k8s针对statefulset, 是如何支持的? 

## 特点
### 网络特性 
依赖HeadlessService, 来为每个Pod分配:
- 一个稳定的, 有序的hostname, hostname命名规则: ${StatefulSetName}-${idx}, ${idx}从0开始
- 一条A记录, 从 ${hostname}.default.svc.cluster.local 到podIp的DNS解析
- 如果是非HeadlessService, 则会生成一条 ${SvcName}.default.svc.cluster.local 到 clusterIp 的DNS解析记录

这样就保证了Pod销毁重建之后: 
- 虽然podIp变化了, 但hostname能跟之前保持一致
- A记录会自动更新, 这样k8s集群内部其他pod通过 ${hostname}.default.svc.cluster.local 仍然能够访问到该pod

### 存储特性: 
是statefulset的特性:     
- 为每个pod创建单独的pvc, 命名规则 ${PvcName}-${StatefulSetName}-${idx}, 并绑定单独的pv, 而不是像deployment为所有pod创建(一个?还是多个?pvc)并绑定同一份pv
- pod销毁重建之后, 由于hostname稳定, 所以还能关联到之前的pv

### 其他顺序化的部署特性:
- Ordered Pod Creation: Pod创建是有序的, 即pod-0先创建, 只有pod-0成功后, pod-1才会创建
- scale-down时, 会按照index倒序删除pod
- 滚动升级(Rolling Update)时, 会按照index倒序删除&重建pod


## statefulset的设计原理模型

- 拓扑状态: 应用的多个实例之间不是完全对等的关系, 这个应用实例的启动必须按照某些顺序启动. 比如应用的主节点A要先于从节点B启动. 而如果你把A和B两个Pod删除掉, 他们再次被创建出来是也必须严格按照这个顺序才行. 并且, 新创建出来的Pod, 必须和原来的Pod的网络标识一样, 这样原先的访问者才能使用同样的方法, 访问到这个新的Pod.
- 存储状态: 应用的多个实例分别绑定了不同的存储数据. 对于这些应用实例来说, Pod A第一次读取到的数据, 和隔了十分钟之后再次读取到的数据, 应该是同一份, 哪怕在此期间Pod A被重新创建过. 一个数据库应用的多个存储实例.

# ZK on K8S
## 背景
为啥 [zookeeper](https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/) 通过k8s的statefulset创建zk服务, 看yaml文件里,
1. 没有创建myid文件
2. 没有在conf里配置 server.$i=$NAME-$((i-1)).$DOMAIN:$SERVER_PORT:$ELECTION_PORT 这种

## 原理
原因: 因为启动zk的脚本是start-zookeeper, 不是start.sh 
所以是因为docker镜像里边对zk进行了封装, 对k8s环境变量进行了适配. 具体查看脚本:
[start-zookeeper](https://github.com/kow3ns/kubernetes-zookeeper/blob/master/docker/scripts/start-zookeeper)

### 如何生成myid文件?
- 依赖statefulset的hostname全局有序且保持稳定的特性
- 在pod内部执行"hostname"命令, 读取hostname, 参见上边hostname命名规则, 读取到${idx}值, myid=${idx}+1

### 如何生成conf文件中的 server.1=xxxxxx:port1:port2 这种peer的记录?
- server.$i=$NAME-$((i-1)).$DOMAIN:$SERVER_PORT:$ELECTION_PORT
- $NAME: 从hostname读取到 ${StatefulSetName}
- $DOMAIN: 从 'hostname -d' 命令中读取到domain信息
