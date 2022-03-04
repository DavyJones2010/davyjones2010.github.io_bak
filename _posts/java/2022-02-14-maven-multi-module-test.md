---
layout: post
title: SpringBoot+Maven多模块项目测试最佳实践
subtitle: SpringBoot+Maven多模块项目测试最佳实践
tags: [springboot, maven, maven-module, maven test]
---

# 背景
项目中为了整体代码的高内聚与可移植性, 按照maven-module的方式对代码进行了拆分, 样例如下: 
```yaml
- root-module
  - shared-common-module
  - middleware-module
    - cache-module
      - cache-api
      - cache-local-impl
      - cache-redis-impl
    - persistence-module
      - persistence-api
      - persistence-mysql
      - persistence-oracle
  - biz-module
    - controller-module
    - service-module
```

# 基本规范
## root-module pom文件有两个作用: 
1. 作为parent module, 管理各个子模块, 便于resolve自身各个模块之间的依赖关系. 如下, biz_module依赖(GAV)middleware-module + shared-common-module; middleware-module依赖(GAV)shared-common-module:

```xml
<modules>
    <module>shared-common-module</module>
    <module>middleware-module</module>
    <module>biz-module</module>
</modules>
```
2. 作为parent pom, 统一管理各个子模块的Maven依赖, 同时默认引入通用的maven依赖(例如apache-commons, guava等), 而无需子模块重复引入.

```xml
<dependencyManagement>
    <dependencies>
    </dependencies>
</dependencyManagement>
<dependencies>
</dependencies>
```
## 子模块之间依赖关系
依赖关系如下: 
```yaml
controller-module --> service-module --> cache-api                              --> shared-common
                                     --> cache-redis-impl/cache-local-impl      --> shared-common
                                     --> persistence-api                        --> shared-common
                                     --> persistence-mysql                      --> shared-common
```


# 问题说明
上述多模块导致的问题, 在SpringBoot场景下, 由于默认的Application是放在最外层的controller-module.  
根据依赖关系, middleware-module, shared-common-module等无法依赖controller-module(否则会导致循环依赖)
所以由于找不到Application入口, 无法by模块地启动spring容器, 进行集成测试.

# 目标
1. 每个模块能做到高内聚, 即模块相关的配置项都放在模块内部. 例如 
   1. persistence-mysql相关的datasource的配置, 放在persistence-mysql模块的resources/xxx.properties里
   2. cache-redis-impl相关的redis配置, 放在cache-redis-impl模块的resources/xxx.properties里
2. 各个模块的测试态/运行态配置项是自说明的, 能便于依赖方进行集成测试, 而无需依赖方再重新配置一份. 例如: 
   1. service-module依赖了 persistence-mysql , cache-redis-impl 两个模块, service-module需要在UT里启动Spring容器, 测试具体service方法. 
   2. 无需把 persistence-mysql里的测试datasource配置, cache-redis-impl里的测试redis配置都在自己的 application.properties 里写一份. 
   3. 可以直接引入对应模块提供的测试properties.
   4. 如果可以实现, 尤其是在模块依赖层次很深很复杂的时候, 使用简便性会更好. 例如 controller-module 层测试, 只需要配置controller层自己的配置就好, 无需关心到底依赖到的是哪些module, 这些module里到底需要哪些配置项.
3. 能够by模块/分层地启动Spring容器进行测试
   1. 方案1: JMockit
      - 通常的一种做法是使用JMockIt, 但带来的问题是, 例如要测试的方法内部实现是调用某个OpenAPI, 可能一开始并不知道结果的结构是怎样的(尤其是在api文档缺失或者不全的情况下), 也不清楚性能怎样. 
      - 这样ut的准确度与可靠性以及后续性能压测的能力都不具备.
   2. 方案2: 参见本文后续.

# 解决方案
## 一: 通用能力放到shared-common里
尤其是log4j.xml等配置, 由于各个模块都需要, 因此需要放入到share-common里.  

## 二: 模块化的配置能力
### 第一步: 模块配置文件按照 通用+测试/生产/等环境相关 两类进行拆分
例如persistence-mysql模块的配置项拆分为如下几个: 
- persistence-mysql-common.properties --> common, 环境无关的配置, 如下:

```
spring.datasource.driver.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.driver.type=com.xxx.druid.pool.DruidDataSource
spring.datasource.driver.maxActive=300
spring.datasource.driver.initialSize=20
```
- persistence-mysql-test.properties --> for ut , 如下: 

```properties
spring.datasource.jdbc-url=jdbc:mysql://xxx:3306/test?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true
spring.datasource.username=usr
spring.datasource.password=pwd
```
- persistence-mysql-dev.properties  --> for dev env
- persistence-mysql-prod.properties --> for prod env

注意: 
这些`*.properties`文件一定都要放在 src/main/resources/ 目录下, 注意尤其不能把  persistence-mysql-test.properties 放到 src/test/resources/ 目录下.
因为如果放到 src/test/resources/ 下, 虽然当前模块自身的UT能加载到 `persistence-mysql-test.properties` 文件
但其他依赖到该模块的UT就无法加载到 `persistence-mysql-test.properties` 文件了.

### 第二步: 使用 spring.profiles.active 对 common + env相关的配置文件 进行加载
```java
@Configuration
@PropertySource({"classpath:persistence-mysql-common.properties", "classpath:persistence-mysql-${spring.profiles.active}.properties"})
public class DataSourceConfig {
}
```

### 第三步: 配置 spring.profiles.active 环境变量
- 方案1: 在UT代码里显式设置环境变量


- 方案2: 通过springboot的默认机制设置环境变量: 
在对应模块的 `src/test/resources/` 目录下新建 `application.properties` 文件, 内容如下:  
```properties
spring.profiles.active=test
```
UT启动时会优先加载 `src/test/resources/application.properties` 文件, 这样就把环境变量设置好了.

### 第四步: UT
```java
@RunWith(SpringRunner.class)
@SpringBootTest
@ActiveProfiles("test")
@Slf4j
public class DefaultServiceTest {
    @Autowired
    DefaultService ds;
}
```

## 三: 模块化的Spring容器测试能力
由于各个模块没有Application.java类的入口, 而SpringBootTest需要有个Application入口.
因此当前的解决方案是 在各个模块的`src/test/java`代码里, 都加入TestApplication.java文件, 作为测试的context入口: 
1. 在父包路径下创建该类, 则无需指定入口类, springboot默认会从 DefaultServiceTest测试类的包路径, 向上查找.
2. 在兄弟路径下设置该类, 则需要指定该入口类名称如下: 
```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {TestApplication.class})
@ActiveProfiles("test")
@Slf4j
public class DefaultServiceTest {
    @Autowired
    DefaultService ds;
}
```

# 思考
其实整体与SpringBoot的分层测试思路很像.

# Refs




