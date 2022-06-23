---
layout: post
title: NAT之端口映射(PAT)原理总结&实践
subtitle: NAT之端口映射(PAT)原理总结&实践
tags: [linux, network, cloud-computing, iaas, nat, pat]
---

# 背景
经常在家用路由器中, 看到有端口映射(PAT)选项, 但不明所以. 
最近刚好在研究NAT, 知道了PAT本质是NAT的一种实现方式, 更确切地说是DNAT的一种实现方式.
因此借此机会好好研究下.

# 总结
## PAT作用概述
家用路由器中配置PAT, 本质上还是`内网穿透`, 即需要把家里内网某台主机的某个服务, 暴露到公网上, 以便能在公网环境下访问该服务.
例如
- 家庭内网网段是`192.168.3.0/24`
- 家用路由器, 电信运营商分配的对外公网IP是`115.192.71.187`
- 某台主机IP是`192.168.3.213`: 
  - 暴露了8080的某个tomcat服务, 想要把这个服务暴露在公网上, 以便其他人能够访问.
  - 暴露了远程桌面RDP端口服务, 以便在公司或者咖啡厅, 能访问到该远程桌面.
  - 等等
- 此时可以在路由器上开启端口映射(PAT), 将路由器上某个端口(如8888)与内网主机的端口(如8080)如进行映射, 所有访问{路由器IP, 8888端口}的请求, 就会被自动路由到{内网IP, 8080端口}上:

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202206232339615.png)

// TODO: 这里挖个坑, 内网穿透除了PAT外, 还有其他方案, 例如 `花生壳`, 后边新开篇章讲下原理.

## 原理概述
本质上是家用路由器保存了这么一张NAT表: 
`{publicIp, publicPort}` ---映射到---> `{localIp, localPort}`

## PAT路由流程
### 0X00: 外部用户 --> 路由器
1. 二层帧: `{remoteMac, remoteIp, remotePort}, {publicMac, publicIp, publicPort}`

### 0X01: 路由器 --> 内网Server:
1. 查NAT表: `{publicIp, publicPort}` --> `{localIp, localPort}`
2. 发ARP广播: what's localIp's mac addr, tell publicMac
3. 获取ARP缓存表项: {localIp} --> {localMac}
4. 修改帧内容如下: 
`{remoteMac, remoteIp, remotePort}, {localMac, localIp, localPort}`

### 0X02: Server端处理:
1. 接收如下IP帧
`{remoteMac, remoteIp, remotePort}, {localMac, localIp, localPort}`
2. 目标 localMac==Server localMac, 网卡接收(无需开启混杂模式), 否则丢弃
3. 目标 localIp == Server localIp, TCP/IP协议栈接收. (TODO: 是否有可能IP不匹配? 如何处理?)
4. server端将TCP数据包转给对应应用层程序处理

### 0X03: Server --回包--> 路由器:
1. Server处理完毕, 生成回包, 由于remoteIp不在网段:
`{localIp, localPort}, {remoteIp, remotePort}`
2. Server发送ARP广播, 查找默认网关(server端事先配置好了网关的IP)的MAC地址 {gatewayIp} --> {gatewayMac}
3. Server封装二层帧, 发送给路由器:
`{localMac, localIp, localPort}, {gatewayMac, remoteIp, remotePort}`

### 0X04: 路由器PAT处理
1. 收到二层帧:
`{localMac, localIp, localPort}, {gatewayMac, remoteIp, remotePort}`
2. 反向查找NAT表: 
`{localIp, localPort} --> {publicIp, publicPort}`
3. 修改三层包(替换掉localIp, localPort):
`{publicIp, publicPort}, {remoteIp, remotePort}`
5. 二层帧修改(修改掉localMac, gatewayMac):
`{publicMac, publicIp, publicPort}, {remoteMac, remoteIp, remotePort}`

### 注意:
路由器本身是有多个网卡的, 内网中是 {gatewayMac, gatewayIp}, 对应的公网是 {publicMac, publicIp}


# 底层原理 
核心是路由器怎么创建NAT表? 使用iptables么? 怎么操作iptables能实现相同功能? 


# 实践&验证 
// TODO: 

# 其他
最近研究关于网络, 有一堆的概念, 一堆的坑要填, 这里先列出来: 
- 几张表, 作用是啥? 分别如何查看表项? 
  - cam表
  - arp缓存表
  - nat表
  - dns缓存表
- 几种设备: 
  - 三层交换机与二层交换机区别?
  - Linux网桥与交换机有啥区别?
- 几个技术细节
  - 网卡混杂模式? 啥时候下开启? 应用场景是啥?
  - 会不会收到MAC地址匹配, 但IP地址不匹配的IP包? 具体咋处理? 直接丢弃? 还是可以使用? 
  - ip forwarding 是啥? 为啥NAT模式下需要开启? 为啥不默认开启? 
  - ip隧道的原理是啥? 
  - lvs的几种工作原理再研究, 再实践验证
  - 组播的应用场景是啥? 很重要么? 
