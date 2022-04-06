---
layout: post
title: k8s基础架构笔记--一些必知的基础数据
tags: [k8s, k8s-architect]
lang: zh
---

# 内存空间占用
- 一个node数据对象的json格式大小约为28KB
- 一个pod的数据对象的json格式大小约为40KB

# 规模
瓶颈主要在watch&list导致apiserver压力巨大上.
- 单cluster 5000个Node, 25000个Pod
