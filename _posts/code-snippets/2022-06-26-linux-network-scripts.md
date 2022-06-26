---
layout: post 
title: 常用的Linux命令之网络相关命令
subtitle: 常用的Linux命令之网络相关命令
tags: [code-snippets, linux, network, arp, nat]
---

# ARP协议
## 查看ARP缓存表

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/PlayGround/nat$ arp -v
Address                  HWtype  HWaddress           Flags Mask            Iface
_gateway                 ether   c0:b4:7d:69:d6:c3   C                     wlp4s0
172.17.0.3               ether   02:42:ac:11:00:03   C                     docker0
192.168.3.2              ether   00:11:32:b2:b7:04   C                     wlp4s0
172.17.0.2               ether   02:42:ac:11:00:02   C                     docker0
Entries: 4	Skipped: 0	Found: 4
```

## 查看某个网卡的arp缓存

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/PlayGround/nat$ arp -i wlp4s0
Address                  HWtype  HWaddress           Flags Mask            Iface
_gateway                 ether   c0:b4:7d:69:d6:c3   C                     wlp4s0
192.168.3.2              ether   00:11:32:b2:b7:04   C                     wlp4s0
```

## 参数详解

- flags:
  - C: complete, each complete entry in the ARP cache will be marked with the C flag.
  - M: 手动增加的，Permanent entries are marked  with  M
  - P: TODO: 不太明白具体啥意思，published entries have the P flag.

# 网桥

## 查看网桥

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242cee53ddd	no		veth9aec769
							            vethe466b6b
virbr0		8000.525400245c90	yes		virbr0-nic
```

## 查看veth pair
如上，发现`veth9aec769`与`vethe466b6b`都挂在`docker0`网桥下。
这两个veth设备都是在host上的，那么两个veth设备对应的pair（分别在2个docker容器中）分别是啥？

1. 查看host上veth设备对应的docker容器内veth设备的编号（如下分别是if23,if24）

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ ip a | fgrep veth9aec769
24: veth9aec769@if23: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 

davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ ip a | fgrep vethe466b6b
22: vethe466b6b@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
```

2. 登录docker容器中查看veth设备编号

如下可知：
- d3f6cb3a496e 容器的veth编号是21，与host上if22，即vethe466b6b绑定
- 333a28ae8ea1 容器的veth编号是23，与host上if24，即veth9aec769绑定

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/PlayGround/docker$ docker exec -it d3f6cb3a496e /bin/sh -c "ip a"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
21: eth0@if22: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
       
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/PlayGround/docker$ docker exec -it 333a28ae8ea1 /bin/sh -c "ip a"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
23: eth0@if24: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

# CAM表（MAC Learning Table）

[How to show a MAC learning table of Linux bridge](https://www.xmodulo.com/show-mac-learning-table-linux-bridge.html)
如下，可以看到docker0网桥（虚拟交换机）的CAM表，即mac addr对应的port no

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ brctl showmacs docker0
port no	mac addr		is local?	ageing timer
  1	22:2f:64:b8:78:af	yes		   0.00
  1	22:2f:64:b8:78:af	yes		   0.00
  2	46:de:b9:ed:19:0f	yes		   0.00
  2	46:de:b9:ed:19:0f	yes		   0.00
```

# 路由
## 查看路由表

```shell
/ $ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      *               255.255.0.0     U     0      0        0 eth0
```
- 这是docker容器内部的路由表：
- 第二条路由规则：从该主机，发往172.17.0.0/16的IP包，都要通过eth0网卡发出（Iface=eth0），不经过网关（Gateway=*）,直接通过二层网络发送过去。
- 第一条路由规则：从该主机，发往其他地址的IP包，都要通过eth0网卡发出（Iface=eth0），要先发给网关172.17.0.1（Gateway=172.17.0.1）


```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.3.1     0.0.0.0         UG    600    0        0 wlp4s0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.3.0     0.0.0.0         255.255.255.0   U     600    0        0 wlp4s0
```
- 这是Host的路由表：
- 第二条路由规则：从该主机，发往172.17.0.0/16的IP包，都要从docker0网卡发出（Iface=docker0），不经过网关（Gateway=0.0.0.0），直接通过二层网络发送过去
- 第三条路由规则：从该主机，发往192.168.3.0/24的IP包，都要从wlp4s0网卡发出（Iface=wlp4s0），不经过网关（Gateway=0.0.0.0），直接通过二层网络发送过去

因此：
- 从主机 `ping 172.17.0.2` 容器，实际是进入了docker0网桥，因此能ping通容器
- 从主机 `ping 192.168.3.2` 即该主机局域网内其他主机，实际是从wlp4s0网卡（物理网卡）出去。


# NAT/iptables



# DNS

## DNS缓存
### Linux上查看DNS缓存
#### 2016年前
[By 2016, Prior to systemd, there was almost no OS-level DNS caching](https://unix.stackexchange.com/questions/28553/how-to-read-the-local-dns-cache-contents)
unless nscd or dnsmasq was installed and running.
Even then, the DNS caching feature of nscd is disabled by default at least in Debian because it's broken. 
[The practical upshot is that your linux system very very probably does not do any OS-level DNS caching.](https://stackoverflow.com/questions/11020027/dns-caching-in-linux)

> If an end user using your software needs to have DNS caching </br>
> because the DNS query load is large enough to be a problem or the RTT to the external DNS server is long enough to be a problem, </br>
> they can install a caching DNS server such as Unbound on the same machine as your application, 
> configured to cache responses and forward misses to the regular DNS resolvers.

#### 2016年后

[After 2016, nowadays on systemd there's a service to cache DNS, it could be enabled with systemctl enable systemd-resolved](https://www.freedesktop.org/software/systemd/man/systemd-resolved.service.html)

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ service systemd-resolved status
systemd-resolved.service - Network Name Resolution
     Loaded: loaded (/lib/systemd/system/systemd-resolved.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2022-05-08 16:04:20 CST; 1 months 18 days ago
       Docs: man:systemd-resolved.service(8)
             https://www.freedesktop.org/wiki/Software/systemd/resolved
             https://www.freedesktop.org/wiki/Software/systemd/writing-network-configuration-managers
             https://www.freedesktop.org/wiki/Software/systemd/writing-resolver-clients
   Main PID: 603 (systemd-resolve)
     Status: "Processing requests..."
      Tasks: 1 (limit: 9305)
     Memory: 2.6M
     CGroup: /system.slice/systemd-resolved.service
             └─603 /lib/systemd/systemd-resolved
```

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
```

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ resolvectl status
Link 3 (wlp4s0)
      Current Scopes: DNS        
DefaultRoute setting: yes        
       LLMNR setting: yes        
MulticastDNS setting: no         
  DNSOverTLS setting: no         
      DNSSEC setting: no         
    DNSSEC supported: no         
  Current DNS Server: 192.168.3.1
         DNS Servers: 192.168.3.1
          DNS Domain: ~.         
Link 6 (docker0)
      Current Scopes: none
DefaultRoute setting: no  
       LLMNR setting: yes 
MulticastDNS setting: no  
  DNSOverTLS setting: no  
      DNSSEC setting: no  
    DNSSEC supported: no  
```

### Chrome上查看DNS缓存

```shell
chrome://net-internals/#dnschrome
```

### Firefox上查看DNS缓存

```shell
about:networking#dns
```

## DNS域名解析排查

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:/etc/systemd$ dig baidu.com
davywalker@davywalker-ThinkPad-X1-Carbon-4th:/etc/systemd$ dig baidu.com +trace
```

# 查看到某个IP的路由信息

在Linux&MacOS上，traceroute命令默认使用UDP，而Windows默认使用ICMP协议。但可以使用`traceroute -I`来强制使用ICMP协议。
[By default Windows tracert uses ICMP and both Mac OS X and Linux traceroute use UDP.](https://serverfault.com/questions/374620/does-traceroute-use-udp-or-icmp-or-both)

```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ traceroute 220.181.38.148
traceroute to 220.181.38.148 (220.181.38.148), 30 hops max, 60 byte packets
 1  _gateway (192.168.3.1)  3.140 ms  3.401 ms  5.204 ms
 2  122.233.112.1 (122.233.112.1)  8.077 ms  8.005 ms  8.613 ms
 3  61.164.3.50 (61.164.3.50)  8.474 ms 61.164.2.2 (61.164.2.2)  8.331 ms  9.131 ms
30  * * *
```

- 使用ICMP协议进行traceroute
```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads$ traceroute -I 220.181.38.148
traceroute to 220.181.38.148 (220.181.38.148), 30 hops max, 60 byte packets
 1  _gateway (192.168.3.1)  3.348 ms  3.300 ms  3.604 ms
 2  122.233.112.1 (122.233.112.1)  46.031 ms  46.236 ms  46.274 ms
 3  61.164.3.50 (61.164.3.50)  8.196 ms  9.610 ms  9.522 ms
 4  * 115.233.18.13 (115.233.18.13)  10.761 ms *
 5  202.97.102.201 (202.97.102.201)  40.430 ms  40.413 ms  40.431 ms
 6  36.110.245.66 (36.110.245.66)  35.611 ms *  34.008 ms
 7  * * *
 8  220.181.182.30 (220.181.182.30)  41.642 ms  41.785 ms  37.416 ms
 9  * * *
10  * * *
11  * * *
12  220.181.38.148 (220.181.38.148)  40.385 ms  40.757 ms  40.478 ms
```

# 其他
路由器上，多个网卡，请求到某个网卡之后，是如何转发到其他网卡的？
例如：
docker容器内部 172.17.0.2，访问host的ip (192.168.3.80)
1. docker容器内部有一条路由规则，非docker容器网段，都到docker0网桥上，docker0网桥IP 172.17.0.1
```shell
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
```

2. 请求到docker0网桥 172.17.0.1 之后，（docker0网桥本身就在Host上），查看路由表，从 wlp4s0 网卡发出（这里具体怎么发出的？）
```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:/etc/systemd$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.3.1     0.0.0.0         UG    600    0        0 wlp4s0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.3.0     0.0.0.0         255.255.255.0   U     600    0        0 wlp4s0
```