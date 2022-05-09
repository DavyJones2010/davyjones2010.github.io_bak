---
layout: post
title: ubuntu初始化配置
subtitle: 记录个人习惯化的ubuntu初始化配置
tags: [ubuntu, system-init, system-config]
---

# 背景
在内网淘了一个 ThinkPad X1 Carbon Gen4 的老古董，
默认是无系统的，也不太想安装毫无设计感，相当丑陋的Windows。
还好家里有Ubuntu的安装U盘，就直接拿来装上去了。
之前整体对Ubuntu的体验还是不错的，无论是从UI与字体设计上，使用流畅度上，软件齐全度上。
本身自己惯用Mac，切换到Ubuntu应该也是无缝的。
实际配置下来，发现软件齐全度比6年前自己使用时要多太多太多了，很开心。
在上边玩儿docker就更方便了，更好地理解并且折腾清楚原理啦。
注意，以下都是基于 Ubuntu 20.04.4 LTS 版本(Focal Fossa)说明的。

# 必要熟悉命令
## apt软件包管理
```shell
# 搜索软件
sudo apt-cache search jdk
# 安装软件
sudo apt-get install openjdk-8-jdk
```

## deb软件包管理
```shell
# 安装.deb软件包
sudo dpkg -i xxx.deb
```


# 必装软件
## 开发软件
* JDK
```shell
sudo add-apt-repository ppa:openjdk-r/ppa
sudo apt-get update
sudo apt-get install openjdk-8-jdk
```

* Maven

* MySQL


* Git

* SourceTree -> GitKraken

* IntelliJ IDEA

* nettools

```shell
sudo apt install net-tools traceroute
-- 安装之后就可以执行 route -n 命令查看路由表信息，执行 traceroute 追踪实际路由信息
```

* Docker
```shell
curl -sSL https://get.daocloud.io/docker | sh
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys xxx
```
* Vim
* Sublime
* Postman
* KubeCtl
* Go


## 效率

* 修改apt软件源

```shell
-- vim /etc/apt/sources.list
deb http://mirrors.163.com/ubuntu/ focal main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ focal-security main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ focal-updates main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ focal-proposed main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ focal-backports main restricted universe multiverse
deb http://ftp.sjtu.edu.cn/ubuntu/ focal main universe restricted multiverse
```

* XMind
* Gliffy Diagrams
* [gnomepomodoro](https://gnomepomodoro.org/)
* ClashX --> QV2Ray
* [坚果云](https://www.jianguoyun.com/s/downloads/linux)

* 截图

```
-- 保存到 ~/Pictures/ 目录下
Alt + PrintScreen # 截取选中的窗口
Shift + PrintScreen # 自由选取

-- 保存到剪贴板
Ctrl + Alt + PrintScreen # 截取选中的窗口
Shift + Ctrl + PrintScreen # 自由选取
```

# 必要配置

### 最大化最小化关闭按钮放到左上角
想要跟Mac习惯保持一致
```shell
gsettings set org.gnome.desktop.wm.preferences button-layout "close,maximize,minimize:"
```
恢复到右上角：
```shell
gsettings set org.gnome.desktop.wm.preferences button-layout ":close,maximize,minimize"
```

先写到这儿吧. 后边有啥再补充.

# 其他配置


## Maven设置阿里云镜像
Copy From [将maven源改为国内阿里云镜像](https://zhuanlan.zhihu.com/p/71998219)

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository/>
  <interactiveMode/>
  <usePluginRegistry/>
  <offline/>
  <pluginGroups/>
  <servers/>
  <mirrors>
    <mirror>
     <id>aliyunmaven</id>
     <mirrorOf>central</mirrorOf>
     <name>阿里云公共仓库</name>
     <url>https://maven.aliyun.com/repository/central</url>
    </mirror>
    <mirror>
      <id>repo1</id>
      <mirrorOf>central</mirrorOf>
      <name>central repo</name>
      <url>http://repo1.maven.org/maven2/</url>
    </mirror>
    <mirror>
     <id>aliyunmaven</id>
     <mirrorOf>apache snapshots</mirrorOf>
     <name>阿里云阿帕奇仓库</name>
     <url>https://maven.aliyun.com/repository/apache-snapshots</url>
    </mirror>
  </mirrors>
  <proxies/>
  <activeProfiles/>
  <profiles>
    <profile>  
        <repositories>
           <repository>
                <id>aliyunmaven</id>
                <name>aliyunmaven</name>
                <url>https://maven.aliyun.com/repository/public</url>
                <layout>default</layout>
                <releases>
                        <enabled>true</enabled>
                </releases>
                <snapshots>
                        <enabled>true</enabled>
                </snapshots>
            </repository>
            <repository>
                <id>MavenCentral</id>
                <url>http://repo1.maven.org/maven2/</url>
            </repository>
            <repository>
                <id>aliyunmavenApache</id>
                <url>https://maven.aliyun.com/repository/apache-snapshots</url>
            </repository>
        </repositories>             
     </profile>
  </profiles>
</settings>
```



# 系统恢复软件
越来越希望能做个类似Windows的Ghost系统，即把现在的系统以及软件以及配置都打包成一个系统镜像，这样后续换电脑，
也能很方便地初始化自己的系统。省得重复安装与配置这么多东西。这个后续研究下吧。
https://blog.csdn.net/leaf6094189/article/details/6009924