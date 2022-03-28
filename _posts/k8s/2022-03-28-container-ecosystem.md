---
layout: post
title: 容器生态的一些研究与对比
tags: [k8s, container, aliyun, aws]
lang: zh
---

# 对比
| 功能  |  阿里云云产品   | aws云产品  | 说明  |
|  ----  |  ----  | ----  | ----  |
| 容器服务 | ECI(Elastic Container Instance)  | ECS(Elastic Container Service) | 提供单纯的容器服务, 不涉及容器的编排等 |
| K8S集群管控服务 | [Dedicated ACK](https://www.aliyun.com/product/kubernetes) | EKS | 可以基于各自的虚拟机实例, 作为Node, 在Node之上生产Pod |
| Serverless | [ASK](https://www.aliyun.com/product/cs/ask)  | Fargate | 在客户侧看到的是一个VirtualKubelet, kube-system空间下的管控组件也都是不可见的 |
| 函数计算服务 | Lambda  | Function Compute |  |
| 混合云扩容方案(虚拟机) | ACK Anywhere  | EKS Anywhere | 本质都是通过可以扩展使用云上虚拟机作为Node |
| 混合云扩容方案(容器) |  Virtual Kubelet | Virtual Kubelet | 本质都是通过VirtualKubelet集成在现有k8s集群中 |

# 思考

## 常见的几种方式

1. 多租户全托管方式(Serverless)
- 例如: 搭建了一个统一的redis/HBase集群, 为不同租户设定不同的namespace, quota等
- 优势: 用户侧接入简单方便, 但定制化的需求满足度与响应度可能不会很好. 
- 缺点: 系统侧运维复杂能力要求较高, 横向扩展能力有可能成为瓶颈.
- 计费方式: 一般都是按量计费, 例如存储的使用量, API的调用量等
- 典型例子: 
  - 大数据场景: 
    - [MaxCompute](https://help.aliyun.com/document_detail/27800.html)
    - [Lindorm Serverless型](https://help.aliyun.com/document_detail/187556.html)
  - 容器场景: [ASK](https://www.aliyun.com/product/cs/ask), 控制平面的组件是多租户共享的.

2. 多租户半托管方式(Hosted)
- 例如: 在阿里云上购买RDS, 选好规格之后, RDS就创建好了, 直接拿到url就能jdbc连接上去了.
- 优势: 搭建成本低, 云产品会提供各种工具方便运维. 例如SQL审计工具, 白屏化的慢SQL查询与优化工具等.
- 缺点: 仍然需要用户感知实例的存在, 需要部分介入运维(例如实例宕机的数据备份恢复等).
- 典型例子: 
  - 数据库场景: [rds](https://www.aliyun.com/product/rds/mysql)
  - 大数据场景: [Lindorm 普通型](https://help.aliyun.com/document_detail/187556.html)
  - 容器场景: [ack](https://www.aliyun.com/product/kubernetes)


3. 单用户完全自建方式(SelfManaged)
- 例如: 自己基于2个ECS, 每个上边都安装了mysql, 搭建了一个主备的mysql集群.
- 优势: 用户侧可管控度极高
- 缺点: 自建初期与后续成本高, 运维成本极高
- 典型例子: 
  - 基于ZK的jar包构建zk集群
  - 基于CDH创建Hadoop集群
  - 等


个人理解, 所有能自建的服务都可以以 `多租户半托管`, `多租户全托管` 方式进行. 但前提是要做好: 
1. ACL鉴权等安全准入机制
2. 多租户的隔离: 数据, 服务QoS, 等
3. 统一运维与版本升级方案
4. 周边系统的支撑:
   1. 管控
   2. 计量计费
   
例如:
- 自建ZK集群, 是否可以改造后支持多租户的接入, 每个用户拿到属于自己的zk地址, 通过zkclient连接上去?


## 自建K8S集群与云上托管K8S集群(例如ACK)有啥区别?
而 自建的k8s集群 与 云上托管的k8s集群, 正如同 多租户半托管 与 单用户完全自建





