---
layout: post
title: ubuntu kvm虚拟化配置
subtitle: 记录ubuntu kvm配置，便于进行虚拟化学习实践
tags: [ubuntu, system-init, system-config, kvm, virtualization]
---

# 环境说明
- OS: Ubuntu 20.04.4 LTS 
- OS Type： 64-bit
- 硬件： ThinkPad-X1-Carbon-4th
- CPU：Intel® Core™ i5-6200U CPU @ 2.30GHz × 4
- MEM：
- Graphics：Mesa Intel® HD Graphics 520 (SKL GT2)
- Disk：

# 步骤
## 1. BIOS开启CPU虚拟化
### 开启
- 开机之后按下F1键
- 进入Security选项
- 进入Virtualization子选项
- Intel(R) Virtualization Technology 选项修改成 Enable
- Intel(R) VT-d Feature 选项修改成 Enable

### 确认
#### 方案一：
执行如下命令，查找到 `Virtualization:                  VT-x` 项目即证明已经开启

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads/Clash-Linux$ lscpu
Architecture:                    x86_64
CPU op-mode(s):                  32-bit, 64-bit
Byte Order:                      Little Endian
Address sizes:                   39 bits physical, 48 bits virtual
CPU(s):                          4
On-line CPU(s) list:             0-3
Thread(s) per core:              2
Core(s) per socket:              2
Socket(s):                       1
NUMA node(s):                    1
Vendor ID:                       GenuineIntel
CPU family:                      6
Model:                           78
Model name:                      Intel(R) Core(TM) i5-6200U CPU @ 2.30GHz
Stepping:                        3
CPU MHz:                         499.999
CPU max MHz:                     2800.0000
CPU min MHz:                     400.0000
BogoMIPS:                        4800.00
Virtualization:                  VT-x
L1d cache:                       64 KiB
L1i cache:                       64 KiB
L2 cache:                        512 KiB
L3 cache:                        3 MiB
```
#### 方案二
执行 `egrep -o '(vmx|svm)' /proc/cpuinfo` 由如下值则为成功
```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~$ egrep -o '(vmx|svm)' /proc/cpuinfo 
vmx
vmx
vmx
vmx
vmx
vmx
vmx
vmx
```

## 2. KVM安装
其中 `libvirt-bin` 为非必要依赖

### 安装

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~$ sudo apt-get install qemu-kvm qemu libvirt-bin virt-manager virt-viewer bridge-utils
```

### 验证

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~$ sudo kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

## 3. 确认环境


# Trouble Shootings

## Your CPU does not support KVM extensions 
```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~$ sudo kvm-ok
Your CPU does not support KVM extensions
```


# Refs

- [开源虚拟化KVM入门（KVM架构+KVM基本管理）视频课程](https://edu.51cto.com/center/course/lesson/index?id=118662)
- [ThinkPad机型BIOS开启VT虚拟化技术](https://blog.csdn.net/ZHOU_VIP/article/details/116767782)
- [ubuntu安装kvm](https://blog.csdn.net/weixin_45761101/article/details/114524806)