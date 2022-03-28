---
layout: post
title: k8s statefulset的研究与实践
tags: [k8s, stateful-set]
lang: zh
---

# 背景
一直很感兴趣, k8s针对statefulset, 是如何支持的? 

## 特点
1. Ordered Pod Creation: Pod创建是有序的, 即pod-0先创建, 只有pod-0成功后, pod-1才会创建
2. unique ordinal index: 
3. a stable network identity: pod删除重建后, hostname保持不变
4. 创建的pvc会与pod持续绑定, 即使pod删除重建, 也会自动绑定上
5. scale-down时, 会按照index倒序删除pod
6. 滚动升级(Rolling Update)时, 会按照index倒序删除&重建pod


# ZK on K8S

为啥 [zookeeper](https://kubernetes.io/docs/tutorials/stateful-application/zookeeper/) 通过k8s的statefulset创建zk服务, 看yaml文件里,
1. 没有创建myid文件
2. 没有在conf里配置 server.$i=$NAME-$((i-1)).$DOMAIN:$SERVER_PORT:$ELECTION_PORT 这种


原因: 是因为启动zk的脚本是start-zookeeper, 不是start.sh, 
所以是因为docker镜像里边对zk进行了封装, 对k8s环境变量进行了适配. 具体查看脚本:
[start-zookeeper](https://github.com/kow3ns/kubernetes-zookeeper/blob/master/docker/scripts/start-zookeeper)



