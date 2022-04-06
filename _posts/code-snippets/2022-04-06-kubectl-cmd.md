---
layout: post 
title: 常用的KubeCtl命令 
subtitle: 常用的KubeCtl命令
tags: [code-snippets, kubectl]
---

# 个人习惯
在`~/.bash_profile`里增加alias: 
```shell
alias k='kubectl'
```

# 资源相关命令
## Pod

### 列出pod(包含所在的node)
```shell
k get pod -n kube-system -o wide
```

### 查看pod详情(包含所在的node)

```shell
k describe pod nginx
```

包含如下重要信息:
- Pod对应的clusterIp
- Pod所在的nodeName
- Pod所属的replicaSet `Controlled By xxx`
- Pod的事件 `Events`
- Pod里container的定义, Volumes的定义, Labels & Annotations

### 登录pod
```shell
k exec -ti <your-pod-name> -n <your-namespace>  -- /bin/sh
```

### 查看pod事件
```shell
[root@iZ2ze8h8q419hf1fze66dkZ ~]# k get events --sort-by=.metadata.creationTimestamp
LAST SEEN   TYPE      REASON    OBJECT              MESSAGE
3m17s       Normal    BackOff   pod/no-annotation   Back-off pulling image "k8s.gcr.io/pause:2.0"
23m         Warning   Failed    pod/no-annotation   Error: ImagePullBackOff
28m         Warning   Failed    pod/no-annotation   Error: ErrImagePull
```

### 查看pod日志
```shell
k logs <your-pod-name> -n <your-namespace>
```

### 删除pod
```shell
k delete pod <your-pod-name> -n <your-namespace>
k delete -f <your-pod.yaml> -n <your-namespace>
```
注意, [<mark>k8s里没有停止Pod的概念, 只有删除(delete)</mark>](https://linuxhint.com/kubectl-stop-pod/)
> Kubernetes does not allow you to stop or pause a pod’s present state and resume it later. No. It is not feasible to pause a pod and restart it at a later time.

这里可以思考下, pod与vm的区别, 为啥vm可以stop, delete的概念? 


## Node
### 获取nodes列表

```shell
k get no/node/nodes -o wide
```
注意, <mark>node没有namespace的概念</mark>

### 查看node详情

```shell
k describe node <your-node-name>
```
包含如下重要信息
- `Non-terminated Pods`: 即所有在当前Node上的pod
- `Capacity`: cpu, mem等, 代表Node整体的.
- `Allocatable`: cpu, mem等, 代表Node去除给kubeproxy/kubelet等系统关键组件预留的资源之后, 可以分配给pod的
- `Allocated resources`: 已经分配出去的cpu, mem等. 按照所有pod的request进行叠加的.

## Service
### 获取service列表
```shell
[root@iZ2ze8h8q419hf1fze66dkZ ~]# k get svc -o wide
NAME            TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)   AGE    SELECTOR
kubernetes      ClusterIP   192.168.0.1       <none>        443/TCP   258d   <none>
nginx-service   ClusterIP   192.168.198.117   <none>        80/TCP    5d8h   app.kubernetes.io/name=proxy
```
注意: 这里的`kubernetes`就是apiserver的clusterIp地址了

### 查看svc详情
```shell
[root@iZ2ze8h8q419hf1fze66dkZ ~]# k describe svc nginx-service
Name:              nginx-service
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app.kubernetes.io/name=proxy
Type:              ClusterIP
IP Families:       <none>
IP:                192.168.198.117
IPs:               192.168.198.117
Port:              name-of-service-port  80/TCP
TargetPort:        http-web-svc/TCP
Endpoints:         172.24.241.16:80,172.24.241.17:80,172.24.241.18:80 + 1 more...
Session Affinity:  None
Events:            <none>
```

### 查看对应的endpoints
```shell
[root@iZ2ze8h8q419hf1fze66dkZ ~]# k get ep
NAME            ENDPOINTS                                                        AGE
kubernetes      10.10.0.162:6443,10.11.0.33:6443,10.9.0.58:6443                  258d
nginx-service   172.24.241.16:80,172.24.241.17:80,172.24.241.18:80 + 1 more...   5d8h
```
注意: 这里的`kubernetes`就是apiserver的pod地址了


### 查看endpoints详情
```shell
[root@iZ2ze8h8q419hf1fze66dkZ ~]# k describe ep nginx-service
Name:         nginx-service
Namespace:    default
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2022-04-01T15:22:45+08:00
Subsets:
  Addresses:          172.24.241.16,172.24.241.17,172.24.241.18,172.24.241.19
  NotReadyAddresses:  <none>
  Ports:
    Name                  Port  Protocol
    ----                  ----  --------
    name-of-service-port  80    TCP

Events:  <none>
```

## 其他

### 获取deployments列表
```shell
k get deploy/deployment/deployments -n <your-namesapce> -o wide
```

### 查看deployments详细信息
```shell
k describe deploy/deployment <your-deploy-name> -n <your-namesapce>
```
包含如下关键信息
- `Replicas`: 副本个数情况
- `Pod Template`: Pod模板的详细信息, 例如label, image等
- `NewReplicaSet`: 对应的rs name信息

### 查看rs列表与详细信息
```shell
k get rs
k describe rs
```


