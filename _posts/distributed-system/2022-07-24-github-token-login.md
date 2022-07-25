---
layout: post
title: 从GitHub CLI禁止用户名密码登录引发的思考与总结
tags: [distributed-system, login, token]
lang: zh
---

# 背景
在本地CLI中push github代码时, 要求输入用户名密码, 但输入密码之后, 提示禁止使用密码登录.
- 根据提示配置了半天SSH免登, 结果发现并不生效, push时仍然让输入账号名密码.
- 后续根据提示, 在[GitHub页面新申请了Token](https://github.com/settings/tokens), 然后使用 用户名+Token 登录就可以了. 
从而引发了诸多疑问与思考.

# GitHub访问几种方式

## 方案1: SSH方式

### 配置方式
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207242155452.png)

这种方式, 可以通过配置SSH免登即可.

### 如何支持不同Host采用不同SSHKey?
如果在本地, 既需要配置gitee的SSH免登, 又需要配置github的SSH免登, 有需要配置其他Host的SSH免登, 怎么能让不同的Host使用不同的公私钥对?

- 配置样例如下: 

```shell
vim ~/.ssh/config
```

```shell
-- 为 git@gitee.com:/xxx ssh地址配置
Host gitee.com
  HostName gitee.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_rsa

-- 为 git@github.com:xxx/xxx ssh地址配置
Host github.com
  HostName github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519

-- 其他地址默认ssh地址配置
Host *
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

- 验证是否配置正确:

```shell
[davywalker@davywalkers-MacBook-Pro ~]$ ssh git@github.com
PTY allocation request failed on channel 1
Hi DavyJones2010! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
```


## 方案2: HTTPS方式

### 配置方式
![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207242155904.png)

这种方式, 即背景中的案例, 必须通过 用户名+Token方式 登录, 即CLI中密码字段, 不要输入账号的密码, 而是输入Token

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207242158549.png)

搜索了下, **从2021年8月13日开始, GitHub已经禁止了 用户名+密码方式 登录**
> From August 13, 2021, <br/>
> GitHub is no longer accepting account passwords when authenticating Git operations. <br/>
> You need to add a PAT (Personal Access Token) instead, <br/>
> and you can follow the below method to add a PAT on your system. <br/>

### GitHub密码存储位置
在使用SourceTree的时候, 由于repo使用的也是HTTPS, 因此也提示输入 用户名+密码, 由于密码方式被禁用, 因此后续再push这个repo, 会一直报禁止密码登录错误.
但也<mark>找不到修改/删除该密码的位置.</mark>

查了下资料, 不同的操作系统, 甚至同样操作系统的不同的版本, 存储方式都不同. 这里以 [MacOS Monterey 12.3.1版本](https://stackoverflow.com/questions/42584934/where-is-my-remote-git-repository-password-stored-on-the-local-machine) 为例:

```shell
davywalkerdeMacBook-Pro:~ davywalker$ git config credential.helper
osxkeychain
```
可以看出来是由keychain进行的管理, 在Mac的 `Applications -> Utilties -> Keychain Access` 

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207252101640.png)

- 可以看到SourceTree应用保存的GitHub Repo密码.
- 同时也可以看到IntelliJ IDEA保存的GitHub Repo密码, 通过显示repo密码发现, 这个密码字段其实就是签发给Intellij的 PAT(Personal Access Token)

## 总结
由于<font color='red'>本地配置的remote repo是HTTPS方式, 因此通过配置SSH免登方式必然是无效的.</font> 

![](https://davywalker-bucket.oss-cn-shanghai.aliyuncs.com/img/202207242202265.png)

# 登录安全思考

## 为啥禁止CLI HTTPS方式通过用户名密码登录?
应该是担心密码泄露. 
但会在哪种情况下泄露密码?
1. 存储过程: 为了防止每次push都重复输入, git client应该把用户名密码存储到本机某个位置了. 
2. 传输过程: HTTPS中间人攻击, 发生概率就较小了
所以应该还是密码存储的风险. 
通过页面登录, 有交互方式可以实现MFA, 但<mark>CLI方式无法进行交互从而实现MFA.</mark>
这样从而减弱了安全性.

## 为啥通过用户名+Token方式登录, 就支持呢
### 几种类型的Token 
- 密码: 时间维度是永久有效, 不可召回. 权限范围是无限的(除非子账号). 可能是有规律的.
- SecretKey: 时间维度通常是永久有效(但支持设定长期), 可以召回. 权限范围是有限的. 通常是UUID等无规律的.
- RefreshToken: 时间维度是较长维度(例如可以60天), 可以召回. 权限范围是有限的. 通常是UUID等无规律的.
- AccessToken: 时间维度是较短维度(例如4个小时), 可以召回(但一般不召回, 通过召回RefreshToken实现). 权限范围是优先的. 通常是UUID等无规律的.

### 几种登录方式
- 方式1: 在网页端, 通常选择密码方式登录, 但需要开启MFA以加固安全. 以该方式作为安全性最强, 权限最大的方式. 
- 方式2: 在服务端SDK里, 通常选择SecretKey方式. SecretKey如果泄露, 可以通过方式1登录, 然后撤销SecretKey的有效性, 重新签发新的SecretKey. 
- 方式3: 在移动端SDK里, 通常会签发一个RefreshToken+AccessToken. 每次AccessToken过期之后, 重新通过RefreshToken调用API申请新的AccessToken. 

因此在GitHub CLI方式登录, 其实就是从方式1(但不带MFA)降级到方案2, 一是限制权限范围, 二是可以随时撤销. 

## 其他解决方案

- 也可以将remote repo切换成[SSH方式](https://docs.github.com/en/get-started/getting-started-with-git/managing-remote-repositories#switching-remote-urls-from-https-to-ssh), 并配置SSH免登实现.