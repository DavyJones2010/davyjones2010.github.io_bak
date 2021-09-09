---
layout: post
title: golang入门笔记之常用基础库
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [golang, golang-lib]
lang: zh
---
go-http 是kube-scheduler healthz, metrics的实现基础

[go-http](https://learnku.com/docs/build-web-application-with-golang/034-gos-http-package-detailed-solution/3171)

---

go-restful 是kube-apiserver的基础, 只用到了它最基础的功能，即路由功能
[go-restful](https://github.com/emicklei/go-restful)
[go-restful-sample](https://hackerain.me/2020/09/28/golang/go-restful-overview.html)

---
Semantic Versioning: 实现语义化版本比较与管理, k8s使用该库进行版本判断
[semver](https://github.com/blang/semver)

---
testify: 用于单测
[testify](http://github.com/stretchr/testify)


