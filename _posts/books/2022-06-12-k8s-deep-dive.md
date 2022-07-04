---
layout: post
title: 深入剖析Kubernetes笔记
cover-img: ["https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206121715361.png"]
tags: [2022, books, k8s, docker, container]
lang: zh
---

很久没有看到这么一本书了, 虽然刚拿到手感觉很薄, 只有不到400页, 价格也贵, 99元. 
比起[k8s权威指南](https://book.douban.com/subject/33444476/)这种大部头, 略显单薄. 
但实际翻看前两章之后, 欣喜万分. 越往后看, 愈发觉得鞭辟入里. 
从底层原理出发, 没有废话, 深入且细致地讲解出docker/k8s实现的各种细节.
有效地解释了自己心中的诸多疑惑. 物有所值. 

--- 
# 一些疑惑解释

## 多进程/富容器

- Q: 如果docker中起了多个进程, 那么在宿主机上是怎样的? 是多个进程 or docker进程中的多个线程?
- A: 在宿主机上是多个进程. 本质上是通过新建进程, 将新进程加入到已经存在的Namespace中实现. 


--- 

- Q: 富容器是如何实现的?
- A: 

--- 
- Q: docker exec 到容器中执行命令, 理论上容器也新开了个进程执行该命令(也就类似富容器了), 是如何实现的? 
- A: 



- Q: 富容器模式有什么缺点? 为啥不推荐使用富容器? 
  - A: 

## 网络模型



## 


