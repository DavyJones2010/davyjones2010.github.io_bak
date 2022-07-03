---
layout: post
title: Linux网络之原理探讨与疑问总结
subtitle: Linux网络之原理探讨与疑问总结
tags: [linux, network, cloud-computing, iaas, nat, pat]
---

# 背景

# FAQ
## 网卡混杂模式

- 啥是网卡混杂模式？
- 正常情况下，在网卡收到二层帧之后，会查看TargetMac是否与自身的Mac地址相同，如果不同，则丢弃该帧。
- 开启了混杂模式之后，即使MAC地址不匹配，也不会丢弃。还是会进入TCP/IP协议栈处理。

- 啥时候需要开启？
- // TODO:


- 如何开启？
```shell
// 开启混杂模式
ifconfig eth0 promisc
// 取消混杂模式
ifconfig eth0 -promisc
// 内核判断网卡是否处于混杂模式是看如下标识，如果置位了0x100，则处于混杂模式
cat /sys/class/net/eth0/flags
```

- 是否开启了混杂模式，就能嗅探局域网（二层网）内的所有数据包了？
- 不一定。
- 针对二层如果是集线器模式，可以嗅探到。
- 但现在大部分二层都是交换机模式。即交换机会根据CAM表，往对应的端口转发，不会无脑地全部端口都转发，因此嗅探不到。


- 开启混杂模式的例子
- 如下，docker0网桥就开启了混杂模式。
- // TODO: 为啥docker0网桥开启混杂模式？具体怎么用的？

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~$ cat /sys/class/net/docker0/flags 
0x1003
```


## 路由器与NAT设备的关联与区别
### 路由器工作原理： 

主要用来联通连个不同的网段，<mark>只修改sourceMac与targetMac，不修改sourceIp与targetIp</mark>

1. 根据传入包指定的targetIp（传入包的MAC地址是路由器当前网卡的MAC地址，IP地址是真正的IP地址）
2. 查询自身路由表，查找到最近的下一跳的IP地址。`route -n`
3. 再根据下一跳的IP地址，查找到对应的MAC地址。arp缓存表
4. 修改数据包的targetMAC地址为下一跳的MAC地址，sourceMAC为当前网卡出口的mac地址
5. 将数据包从对应网卡送出去。

> 路由器的核心是路由表的维护，能生成最短最优路径。<br/>
> 从sourceIp到targetIp的路由路径，很可能与回包，即从targetIp返回到sourceIp的路由路径不一样。
> 取决于当时的路径中路由器的路由表情况。

### NAT工作原理：

- 原理
需要修改sourceMac与targetMac。
同时如果是SNAT，则还要修改sourceIp。 
如果是DNAT，则还要修改targetIp。

- 实现方式:
通过iptables可以设置nat表。
snat作用在postrouting阶段，dnat作用在prerouting阶段。
回包由内核的conntrack模块负责，不再由iptables负责。
详细参见 [NAT之端口映射(PAT)原理总结&实践](https://davyjones2010.github.io/2022-06-23-linux-network-nat)


### 总结
由此可知，家用的路由器，是附带了SNAT（也可以配置DNAT）功能的路由器。



## ip_forwarding 
### MAC地址匹配, 但IP地址不匹配
网卡会不会收到MAC地址与当前网卡MAC地址匹配, 但IP地址与当前网卡的IP不匹配的IP包?
具体咋处理? 直接丢弃? 还是可以使用?

- 会收到。
- 如果开启了ip_forwarding，则会进入FORWARD/ROUTE阶段，查看本地 route 表，进入路由与postrouting阶段，找到对应的出口网卡，将包转发出去。与路由器的功能完全一致了。
- 如果未开启ip_forwarding，则直接丢弃该包。
- 所以，NAT模式下，必须要开启ip_forwarding。

此图摘自 [Archlinux 文档](https://wiki.archlinux.org/title/Iptables)
```
                               XXXXXXXXXXXXXXXXXX
                             XXX     Network    XXX
                               XXXXXXXXXXXXXXXXXX
                                       +
                                       |
                                       v
 +-------------+              +------------------+
 |table: filter| <---+        | table: nat       |
 |chain: INPUT |     |        | chain: PREROUTING|
 +-----+-------+     |        +--------+---------+
       |             |                 |
       v             |                 v
 [local process]     |           ****************          +--------------+
       |             +---------+ Routing decision +------> |table: filter |
       v                         ****************          |chain: FORWARD|
****************                                           +------+-------+
Routing decision                                                  |
****************                                                  |
       |                                                          |
       v                        ****************                  |
+-------------+       +------>  Routing decision  <---------------+
|table: nat   |       |         ****************
|chain: OUTPUT|       |               +
+-----+-------+       |               |
      |               |               v
      v               |      +-------------------+
+--------------+      |      | table: nat        |
|table: filter | +----+      | chain: POSTROUTING|
|chain: OUTPUT |             +--------+----------+
+--------------+                      |
                                      v
                               XXXXXXXXXXXXXXXXXX
                             XXX    Network     XXX
                               XXXXXXXXXXXXXXXXXX
```

### 为啥Linux默认不开启ip_forwarding?
因为在默认情况下，host只是扮演在网络中的一台主机的角色，根本不需要ip_forwarding的功能。

> A Linux machine acting as an ordinary host would not need to have IP forwarding enabled, 
> because it just generates and receives IP traffic for its own purposes

### 如何开启ip_forwarding?

```shell
echo 1 > /proc/sys/net/ipv4/ip_forward
```

### 总结
所以开启了ip_forwarding，相当于本机就具备一台路由器的能力了。<br/>
如果再通过iptables配置了相应的nat规则，就相当于一台家用的路由器了。

## IP隧道


## LVS的几种模式分析


