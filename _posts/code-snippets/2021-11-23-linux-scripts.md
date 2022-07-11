---
layout: post 
title: 常用的Linux命令
subtitle: 常用的Linux命令
tags: [code-snippets, linux]
---

# 背景
# 常用Linux命令

## 文本处理
* 文件按固定行数切分

```shell
split -l 20 cn-hangzhou.csv cn-hangzhou-
```

问题是: 生成的文件都是没有后缀名的, 如下可以使用xargs, 统一增加后缀名, 如下：
- mac:

```shell
split -l 20 cn-hangzhou.csv cn-hangzhou- && ls | grep cn-hangzhou- | xargs -n1 -I{} mv {} {}.txt
```

- linux:

```shell
split -l 20 cn-hangzhou.csv cn-hangzhou- && ls | grep cn-hangzhou- | xargs -n1 -i{} mv {} {}.txt
```

* 文件按照某列进行排序

```shell
sort -n -k 1 -t , sample.csv
```

* 文件查看某列去重数据

```shell  
awk -F',' '{print $1}' sample.csv | sort -n | uniq
```

* 计算文件中相同的行数量, 并按照数量倒序排列:

todo: 

* grep正则表达式
  grep正则表达式元字符集（基本集）
  ^锚定行的开始 如：'^grep'匹配所有以grep开头的行。
  $锚定行的结束 如：'grep$'匹配所有以grep结尾的行。

* JSON处理

```shell
jq
```

* 创建空的大文件(全被0占据的文件, 而非打洞)

```shell
fallocate -l 50M /data/web/www/html/b.zip
## -l: 指定文件的长度
```

## 目录管理

* 查看目录大小

```shell
sudo du -h --max-depth=1 /home/admin/
```

* 查看目录下文件大小

```shell
du -sh * | sort -nr | head
du -sh * | sort -nr | fgrep "G" | head
```

## 系统管理
### top
#### 进程排序
- 以 CPU 占用率大小的顺序排列进程列表 
  - top进入之后， 按P 
  - 或者 `top -o %CPU`
- 以内存占用率大小的顺序排列进程列表
  - top进入之后，按M 
  - 或者 `top -o %MEM`
- 以总共消耗的CPU时间片排序
  - top进入之后，按T 
  - `top -o +TIME`

#### 线程排序
- 查看某个进程下各个线程CPU占用情况，倒序排列
  - `top -Hp ${pid} -o %CPU`
- 查看某个进程下各个线程内存占用情况，倒序排列
  - `top -Hp ${pid} -o %MEM`
- 查看某个进程下各个线程CPU时间片占用情况，倒序排列
  - `top -Hp ${pid} -o +TIME`

### centOS yum源路径

```shell
/etc/yum.repos.d/
```

## 进程&网络管理

* 查看端口被哪个进程占用了

```shell
lsof -i:${port}
netstat -natp | fgrep ${port}
```

* 根据进程号, 查看进程占用了哪些tcp端口

```shell
netstat -natp | fgrep ${pid}
sudo lsof -i -P | fgrep ${pid}
```

* 查看进程信息:

```shell
ps aux | fgrep ${pid}
```

## 用户管理
* 锁定用户:
```shell
sudo passwd -l username
```

* 查看用户是否被锁定:

```shell
sudo passwd -S username
sudo passwd --status interlive
```

- 或者看下密码前边是否有 "!!" 标记, 如果有, 则证明被标记了

```shell
sudo fgrep "interlive" /etc/shadow
```

## 绑核
- 查看进程绑核情况

```shell
taskset -p $PID
taskset -cp $PID
```
```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads/Clash-Linux$ taskset -p 20697
pid 20697's current affinity mask: f
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads/Clash-Linux$ taskset -cp 20697
pid 20697's current affinity list: 0-3
```

- 执行进程绑核操作

```shell
taskset -p COREMASK PID
```