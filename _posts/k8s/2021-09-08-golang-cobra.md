---
layout: post
title: golang入门笔记
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [golang, cobra]
lang: zh
---
# 目标
- 创建简版的ECS CLI, 实现实例创建等基础操作
```shell
runInstances --region cn-beijing --zone cn-beijing-h --instanceType ecs.c5.large --payType spot --image xxx --vpc xxx --vsw xxx --sg xxx --count 1
```

- 创建go版本 [Preemptive Instance Recommendation CLI](https://github.com/aliyun/alibabacloud-ecs-easy-sdk/tree/master/incubator-plugins/preemptive-instance-recommendation)

- 创建&部署[spot-interruption-handler](https://github.com/kube-aws/kube-spot-termination-notice-handler)


# cobra
## 参考文档
[在 Golang 中使用 Cobra 创建 CLI 应用](https://www.qikqiak.com/post/create-cli-app-with-cobra/)

## 常用命令

### 初始化cobra CLI应用脚手架
```shell
cobra init --pkg-name spot-tool
```

### 编译二进制
```shell
go build -o spot-tool
```

### 增加子命令
```shell
增加子命令
cobra add runInstances
```

```shell
执行子命令 ./rootCmd subCmd params
./spot-tool runInstances 10 11 12 13 14
```

### 增加孙命令
```shell
增加子命令
cobra add subRunInstances
```

```shell
修改subRunIntances.go的init
rootCmd.AddCommand(subRunInstancesCmd) 改成
runInstancesCmd.AddCommand(subRunInstancesCmd) 
```

```shell
执行孙命令
./spot-tool runInstances subRunIntances params
```


### 为命令增加flag
```shell
子命令init中增加标识 
runInstancesCmd.Flags().BoolP("float", "f", false, "Add Floating Numbers")
```

```shell
Run: func 中增加识别该标识的逻辑
fstatus, _ := cmd.Flags().GetBool("float")
if fstatus {
    floatAdd(args)
} else {
    intAdd(args)
}
```

### k8s中如何使用cobra


