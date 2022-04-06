---
layout: post
title: 云计算笔记--aws的DefaultVpc实现调研
tags: [cloud, iaas, aws, default-vpc, vpc]
lang: zh
---

[AWS Default VPCs](https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html)
1. 默认每个账号下, 每个地域下, AWS都会自动创建 172.31.0.0/16  CIDR的VPC
2. 会在账号下Default VPC内, 每个可用区都创建一个VSW, 范围是:
   a. VSW网段范围: [172.31.00000000.0/20, 172.31.11110000.0/20] 共16个VSW
   b. 每个VSW内, IP数量: 2^12=4096 个
3. 新的可用区上线之后, 等待几天, AWS就会自动创建对应可用区的Default VSW.
4. 只要对默认VPC进行了配置修改, 那么新的可用区上线之后, 就不会为之自动创建DefaultVSW了. 

