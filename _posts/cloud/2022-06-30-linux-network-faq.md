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

### iptables工作原理图
此图摘自 [Archlinux 文档](https://wiki.archlinux.org/title/Iptables)
```
                               XXXXXXXXXXXXXXXXXX
                             XXX     Network    XXX
                               XXXXXXXXXXXXXXXXXX
                                       +
                                       |
                                       v
 +-------------+              +------------------+
 |table: filter| <---+        | table: nat/dnat  |
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
+--------------+      |      | table: nat/snat   |
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

## 本地多网卡curl公网地址问题
- 如果本机有私网网卡virbr0, 公网网卡en0, 且开启了virbr0网段的SNAT规则. 则在本地curl公网地址, 则curl的出口网卡是啥? 
- 是通过virbr0, 然后走SNAT到en0出去么?
- 还是直接通过en0出去?
1. curl 支持指定出口网卡如下(实际只是使用了对应网卡的ip地址作为SIP): 如果不指定, 则使用默认网卡. 
```shell
// curl指定网卡出口
curl --interface virbr0 http://www.baidu.com
// mac上查看默认网卡: 
route -n get 0.0.0.0 | grep interface
```
> The --interface options is used to figure determine what address on the system will be used as the source IP. <br/> 
> It doesn't magically change anything about routing.

2. 参见[iptables工作原理图](#iptables工作原理图), 可知从curl公网地址, 就是从local process出去, 接下来会走route, 即查看Host的路由表, 确定出口网卡与网关信息.
3. 接下来进入postrouting(SNAT)阶段, 如果配置有SNAT规则, 且刚好命中, 则执行SNAT.
4. 所以结论是出口网卡是en0, 而不会经过virbr0(即使指定了`--interface virbr0`).
5. 是否会SNAT, 取决于是否指定了`--interface virbr0`, 指定了, 则会走SNAT; 未指定, 则不需要走SNAT.

## 本地多网卡Java应用获取HostIp问题
- 如果本机有多块网卡, 在Java中如何获取到这些IP地址? 
- 我们通常使用获取本机IP的方式如下: <mark>当Host多块网卡时, 不好兼容, 存在严重缺陷, 尤其是在线上服务器网卡堆叠场景下!</mark> 

```java
InetAddress ip = InetAddress.getLocalHost();
String hostname = ip.getHostName();
```

- 解决方案: 
- 方案1: 使用 `NetworkInterface.getNetworkInterfaces()` 接口, 获取所有网卡, 然后逐个获取IP信息.
- 方案2: 获取本机的FQDN, 然后使用`InetAddress.getByName(FQDN)`来获取主网卡的IP信息.
  - 本机的FQDN可以写在配置文件里.
- Ref: [How to get Server IP Address and Hostname in Java](https://crunchify.com/how-to-get-server-ip-address-and-hostname-in-java/)

## IP隧道


## LVS
### LVS v.s. iptables/netfilters
本质上, 两个都可以做四层的DNAT, 都是工作在内核空间. 

- LVS: 更侧重于LoadBalancing, 实现上使用hash查表, 而不是iptables的顺序查询. 效率更高. 
- iptables: 

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207042336674.png)
- 可以看到, LVS本质上是与iptables配合使用
- 客户访问虚拟 IP（VIP）时，数据包先在主机内核空间被 PREROUTING 链检测，根据数据包的目标地址进行路由判断，若目标地址是本地，则交由 INPUT 链进行处理。
- IPVS 工作于 INPUT 链，当数据包到达 INPUT 链时，会先由 IPVS 进行检查，并根据负载均衡算法选出真实服务器 IP。
- IPVS 转发模式为 NAT 模式时，将数据包由 FORWARD 链进行处理后由 POST-ROUTING 链发送给真实服务器。
- IPVS 转发模式为非 NAT 模式时，则将数据包由 POST-ROUTING 链发送给真实服务器。

### LVS的几种模式分析
#### LVS-NAT
类似iptables的端口映射(PAT)
- Client请求: {CIP, DIP}
- LVS映射: 修改目标IP, DIP->RIP, 构建新的请求: {CIP, RIP}
- Server处理: 完成后发送回包, {RIP, CIP}; 由于CIP与RIP不在同一个网段, 因此发送到网关(即LVS)
- LVS映射: 修改源IP, RIP->DIP, 构建新的响应: {DIP, CIP}

#### LVS-DR


#### LVS-TUN

#### 实际
- [阿里云CLB](https://help.aliyun.com/document_detail/27544.html)使用的是哪种模式? NAT?
  - 四层采用开源软件LVS（Linux Virtual Server）+ keepalived的方式实现负载均衡
  - LVS应该使用的是NAT模式. 但如果是NAT模式, 则必须把LVS作为ECS的网关. //TODO: 待验证


# HTTP 代理

# 什么是"冷土豆路由"与"热土豆路由"?
- 参见: [冷土豆路由&热土豆路由](https://docs.microsoft.com/zh-cn/azure/virtual-network/ip-services/routing-preference-overview)
- 热土豆:
  - 入口流量: 如果来自新加坡的用户访问托管在芝加哥的 Azure 资源，则流量将通过公共 Internet 传输，并进入芝加哥的 Microsoft 全球网络。
  - 出口流量: 出口流量遵循相同的原则。 流量会在托管服务的同一区域退出 Microsoft 网络。 例如，如果来自 Azure 芝加哥服务的流量最终传输给来自新加坡的用户，流量将离开芝加哥的 Microsoft 网络，并通过公共 Internet 传输给新加坡的用户。
- 冷土豆:
  - 入口流量: 如果来自新加坡的用户访问托管在美国芝加哥的 Azure 资源，则流量将进入位于新加坡 Edge POP 的 Microsoft 全球网络，并通过 Microsoft 网络传输到托管在芝加哥的服务。
  - 出口流量: 如果来自 Azure 芝加哥的流量最终传输给来自新加坡的用户，那么流量就会通过 Microsoft 网络从芝加哥传输到新加坡，并退出位于新加坡 Edge POP 的 Microsoft 网络。
  