---
layout: post
title: k8s基础架构笔记--list&watch机制
tags: [k8s, k8s-architect, list-watch]
lang: zh
---

本质上:  
- list: 就是一个GET的HTTP操作, 请求到apiserver, 是个短连接. list结束, 连接就关闭.
- watch: 是一个基于`Transfer-Encoding: chunked`的[HTTP长连接](https://imququ.com/post/transfer-encoding-header-in-http.html), apiserver作为http server, kube-proxy/kubelet/kube-scheduler等作为http client. 

这样: 
- list: 来实现kube-proxy, kubelet, kube-scheduler等角色启动初始化时, 存量数据的一次拉取.
- watch: 来实现后续各种资源的增量更新

扩展: 
- watch: 
  - k8s java client中watch机制, 是基于OkHttpClient. // TODO: 补上具体实现例子
  - watch机制也可以基于websocket来实现
