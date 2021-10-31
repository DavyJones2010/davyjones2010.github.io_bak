---
layout: post
title: 2021年10月第五周
cover-img: ["/assets/img/20211031-code83.png", "/assets/img/20211031-code83-2.png"]
tags: [weekend, diary, books]
lang: zh
---
第一次完整地参加了一次代码打大赛, 即阿里云举办的 [83行代码](https://code83.ide.aliyun.com/billboard). 
整体没有特别投入, 所以最终成绩也不算特别好. 

---
第二题: 一看题目就想到使用TierTree, 但整体优化思路跟Perf工具没有特别好. 最主要是内存占用优化绕了圈子, 最终也没有走出来. 个人对最终结果不是特别满意.
具体实现在这里: [round2](https://github.com/DavyJones2010/round2.git)

---
第三题: 基本面向对象的重构完全没问题. 但正确率一直在50分. 始终想不到哪里的逻辑出错了. 
主要一方面原因可以归结于题目描述过于含糊, 感觉成了阅读理解. 等待最终结果公布再看下吧.

具体实现在这里: [round3](https://github.com/DavyJones2010/round3.git)

--- 
第四题: 时间比较紧张, 但基本思路还OK. Bug基本也修复了. 
1. SpringSecurity禁用掉CSRF校验
2. addUser时, 使用的admin账号的密码有误, 使用了基础的BaseAuthentication方式, 账号&密码是用base64编码的. 修复掉初始化时的admin密码即可. 
3. 权限设置也有点问题. 很简单就修复了. 
4. 根据协议反序列化为String时, 不能把编码方式放在ThreadLocal里. 
5. DataBuffer转化为byte[]是, 细节处理的不太好. 
不过最终算是把乱码问题也都修复了. 在这个过程查漏补缺, 发现个人对DataBuffer不是特别熟悉, 找机会恶补一波.

---
总结, 整体还是比较顺利的, 个人也真切感受到了其中的迷茫紧张与乐趣, 也得到了一些技术上的进步. 

最终虽然只集齐了8个线索, 但基本满足预期, 可以另一个限量版的公仔哇咔咔.

总体来说作为一个工作多年的老司机, 被这么多后辈成绩甩在后边, 真是心有不甘, 后生可畏, 继续努力!

后续有这种活动, 还要继续参与, 简直是小投入, 大回报~
