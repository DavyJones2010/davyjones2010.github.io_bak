---
layout: post
title: Java与Spring中SPI机制的探讨与思考
tags: [java, spring, spi, service-loader]
---

# 背景
项目中用到了SPI机制, 使用过程也走了一些弯路. 这里记录下来. 

## 几种SPI方式

## Java中SPI机制

### 使用方式

### 实际应用场景

- jdbc驱动, 例如mysql-connector-java.jar中, 就在META-INF/services/java.sql.Driver


### 问题
比较严重的一个问题是, 这些SPI的Bean游离于Spring容器外. 
SPI中引入的对象(Bean)没有被Spring容器管理起来. 
从而这些Bean如果依赖到了Spring容器中其他Bean, 就只能手动从Spring容器中拿出来复制, 即手动注入. 
太麻烦了. 

## Service Loader Spring integration

## Spring Factories Loader

# 思考

## 在什么时候需要用到ServiceLoader/SPI? SPI优点具体是啥? 
个人理解, 最大的优点是编译期与实现类/包解耦. 

## Spring Factories Loader在Starter的应用场景, 优势是啥?









# Refs
[Java Service Loader vs. Spring Factories Loader](https://dzone.com/articles/java-service-loader-vs-spring-factories-loader)