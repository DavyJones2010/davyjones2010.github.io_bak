---
layout: post
title: k8s virtual-kubelet的实现原理研究
tags: [k8s, virtual-kubelet, vk]
lang: zh
---

# 背景
一直很感兴趣, vk的实现原理, 看官方的文档, 太偏向用户视角, 
研究了源代码, 工作机制先大概说下: 
1. 使用apiserver的admission插入点机制, vk启动了一个webserver, 当匹配到vk时, 会将请求路由到vk的webserver
2. vk的webserver做简单的一件事: 将pod的nodename修改成自己的
3. 由于以及有了nodename, 因此跳过sheduler过程, bind请求直接发到对应的kubelet.
4. 这里的kubelet就是vk启动的虚拟kubelet. 
5. vk收到bind请求, 调用云厂商的OpenAPI, 创建pod. 

vk本质上是以pod形式运行在用户自己的k8s集群上.

# 总结
所以vk做了几件事: 
1. 负责维护pod的生命周期, 例如pod的创建与销毁.

其他思考:
1. pod的心跳检查与事件上报链路是怎样的?
2. pod的configmap, secret等信息更新, 如何推送到pod上?
   1. 是vk推送? 
   2. or pod上安装agent主动pull?


  