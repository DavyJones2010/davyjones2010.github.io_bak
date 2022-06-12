---
layout: post
title: Docker的一些总结II
tags: [k8s, docker, container]
lang: zh
---


# exec到容器的原理
本质是在host上新创建一个进程, 加入到已存在的docker容器的Linux Namespace中.

## 1. 查看已有docker容器的namespace信息

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122047763.svg)

```shell
docker ps
docker inspect --format '{{ .State.Pid }}' dockerContainerId
sudo ls -l /proc/$pid/ns
```

## 2. 通过setns()系统调用, 将新进程加入到已有docker容器的namespace中

核心代码如下: 

```shell
int main(int argc, char *argv[]){
	int fd;
	fd = open(argv[1], O_RDONLY);
	if(setns(fd, 0)==-1){
		errExit("setns");
	}
	execvp(argv[2], &argv[2]);
	errExit("execvp");
}
```

```shell
./set_ns /proc/$pid/ns/net /bin/sh
```
- argv[1]: 即当前新进程要加入的Namesapce文件路径
- argv[2]: 即新进程要在Namesapce里运行的程序


## 3. 验证
如下样例, 通过exec进入容器, 在容器内编写并运行了一个`tmp.sh`脚本. 
可以看到, <mark>实际上是在host上新建了一个进程, 且该进程与docker容器共享同一份namespace.</mark>

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122112834.svg)

# volume的原理

- 允许将宿主机上指定的目录或者文件挂载到容器中读取和修改

```shell
docker run -v /test myimageid
docker run -v /home:/test myimageid
```

- 第一种方式, 没有显式声明宿主机目录, Docker默认在宿主机上创建一个临时目录`/var/lib/docker/volumes/[volume_id]/_data`, 然后把它挂载到容器的`/test`目录
- 第二种方式, 把宿主机的`/home`目录挂载到Docker的`/test`目录下

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122150473.svg)

- 这里用到的挂载技术, 就是<font color='red'>Linux的绑定挂载(bind mount)机制</font>. <mark>允许将一个目录或者文件, 而不是整个设备挂载到指定目录上.</mark>
- 并且在挂载点上进行的任何操作, 只是发生在被挂载的目录或者文件上, 而原挂载点的内容会被隐藏起来, 且不受影响.


## bind mount 样例

- 将 `test2/` 目录挂载到 `test/` 目录(挂载点)下, 之后所有在`test/`下的修改在`test2/`下都能看到, 反之亦然.

```shell
sudo mount --bind test2 test
```

- 将 `test2/` 从 `test/` 目录(挂载点) 卸载掉, 之后`test/`维持一个空的挂载点, 所有变更都落到了`test2/`下

```shell
sudo umount test
```

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122208304.svg)


# 单机容器网络原理

## 原理概述

> 本质是通过 docker0网桥 + VethPair 实现单机间多个docker容器互联

- docker0网桥 工作在数据链路层. 类似一个虚拟交换机. 维护 CAM表(交换机通过MAC地址学习维护的 端口与MAC地址的对应表)
- 各个容器通过 VethPair 与docker0网桥连接
- 从host->container, 路由规则, 需要通过 docker0 网桥; docker0网桥查询CAM表, 直接把请求转发到相应端口即可.
- 从container -> container, 通过 veth pair 到达 docker0 网桥; docker0网桥查询CAM表, 直接把请求转发到相应端口即可.

## 常用命令

- 查看网桥信息

```shell
brctl show
```

- 查看路由表

```shell
route
```


## 实例演示

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122305851.png)
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122305137.png)
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122305508.png)
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206122305564.png)






