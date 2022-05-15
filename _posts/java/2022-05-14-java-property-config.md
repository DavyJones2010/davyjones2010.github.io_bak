---
layout: post
title: Java中配置文件占位符替换的几种模式探讨与分析
tags: [java, config, properties, best-practice]
---

# 背景
在Java/Maven项目中, 经常会有配置文件中配置项, 需要根据环境进行不同的参数变量替换, 如果搞不清楚, 经常会混淆.
当前有如下几种方式: 
- 编译期替换
  - Maven Profile
  - [autoconfig](https://juejin.cn/post/6844903962240352263)
- 运行期替换
  - SpringBoot Profile
- 其他替换
  - Docker Java应用启动参数

各有优缺点. 

# 1. 编译期替换
## 1.1 Maven Profile
### 使用方式
- 参见[Maven Profile使用方式](https://www.cnblogs.com/davenkin/p/advanced-maven-use-profile.html)
#### 方式1: 默认激活
```xml
<profiles>
  <profile>
    <id>apple</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <properties>
      <fruit>APPLE</fruit>
    </properties>
  </profile>
  <profile>
    <id>banana</id>
    <properties>
      <fruit>BANANA</fruit>
    </properties>
  </profile>
</profiles>
```

```shell
mvn initialize
mvn clean compile -DskipTests
```

#### 方式2: 指定profile激活

```shell 
mvn initialize -Pbanana
mvn clean compile -DskipTests -Pbanana
```

#### 方式3: 自动激活Profile
如下根据maven内置的系统变量进行, [更多系统变量](https://maven.apache.org/enforcer/enforcer-rules/requireOS.html)
```xml
<profile>
   <id>mac</id>
   <activation>
       <activeByDefault>false</activeByDefault>
       <os>
           <family>mac</family>
       </os>
   </activation>
   <properties>
       <database.driverClassName>org.postgresql.Driver</database.driverClassName>
       <database.url>jdbc:postgresql://localhost/database</database.url>
       <database.user>username</database.user>
       <database.password>password</database.password>
   </properties>
</profile>
<profile>
   <id>unix</id>
   <activation>
       <activeByDefault>false</activeByDefault>
       <os>
           <family>unix</family>
       </os>
   </activation>
   <properties>
       <database.driverClassName>com.mysql.jdbc.Driver</database.driverClassName>
       <database.url>jdbc:mysql://localhost:3306/database</database.url>
       <database.user>username</database.user>
       <database.password>password</database.password>
   </properties>
</profile>
```

### 配置文件变量替换
如下, 根据Maven Profile来选择不同的filter文件, 根据不同的filter文件, 编译期将 `src/main/resources/*` 中的变量都替换掉. 

```xml
<project>
...
<build>
    <filters>
        <filter>src/main/filters-${active.profile}.properties</filter>
    </filters>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <active.profile>dev</active.profile>
        </properties>
        <!-- 把当前profile设置为默认profile，可以同时这是多个为默认-->
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
    <profile>
        <id>test</id>
        <properties>
            <active.profile>test</active.profile>
        </properties>
    </profile>
    <profile>
        <id>product</id>
        <properties>
            <active.profile>product</active.profile>
        </properties>
    </profile>
...
</project>
```

## 1.2 autoconfig
本质其实与maven filter类似. 不做过多解释. 参见: [Webx使用Autoconfig总结](https://juejin.cn/post/6844903962240352263)


# 2. 运行期替换
## 2.1 SpringBoot Profile
初步使用方式参见: [使用Profiles](https://www.liaoxuefeng.com/wiki/1252599548343744/1282388483112993)
具体多模块化多层次配置参见: [SpringBoot+Maven多模块项目测试最佳实践](https://davyjones2010.github.io/2022-02-14-maven-multi-module-test/#%E7%AC%AC%E4%B8%80%E6%AD%A5-%E6%A8%A1%E5%9D%97%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E6%8C%89%E7%85%A7-%E9%80%9A%E7%94%A8%E6%B5%8B%E8%AF%95%E7%94%9F%E4%BA%A7%E7%AD%89%E7%8E%AF%E5%A2%83%E7%9B%B8%E5%85%B3-%E4%B8%A4%E7%B1%BB%E8%BF%9B%E8%A1%8C%E6%8B%86%E5%88%86)
优点: 无需重新编译打包, 直接指定不同的运行参数即可. 


# 3. 其他形式替换
如上, Maven Profile方案, 如果需要dev/pre/prod不同环境打出来不同的包, 需要在maven编译时手动指定 `-Pdev`.
但在实际CI/CD中, 不可能手动敲命令来打包/指定参数. 因此都需要与CI/CD框架进行结合, 从而感知到不同环境参数. 
## 3.1 自定义部署脚本
这个依赖于CI/CD框架本身. 
- 在Jenkins里, 可以手动为不同的pipeline(环境), 指定不同的命令. dev pipeline指定`-Pdev`
- 在其他框架里, 手动编写启动脚本等, 识别CI/CD框架的环境变量来指定`-Pdev`等参数

## 3.2 Docker Java应用启动参数
本质还是在Dockerfile中, 与CI/CD框架结合. 
### 3.2.1 Dockerfile里指定环境变量, 直接在 Spring Properties文件里使用
例如: [Dockerfile: Environment Variables](https://charith.xyz/docker/dockerfile-environment-variables/)
```properties
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```

```shell
FROM maven:3.6.3-openjdk-16-slim AS build
ENV DB_HOST=db \
     DB_USERNAME=username \
     DB_PASSWORD=password
COPY ./mobile-app-ws /usr/local/mobile-app-ws
WORKDIR /usr/local/mobile-app-ws/
RUN mvn -Dmaven.test.skip=true clean package

FROM tomcat:8
COPY --from=build /usr/local/mobile-app-ws/target/mobile-app-ws-0.0.1-SNAPSHOT.war /usr/local/tomcat/webapps/mobile-app-ws.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

基于不同的参数启动镜像: (注意ENV变量不在构建镜像时生效) 
```shell
docker run -d --name ws-myappv3 \
   -e DB_HOST=db-myapp \
   -e DB_USERNAME=charith \
   -e DB_PASSWORD=charith \
   -p 38083:8080 \
   --network=myapp \
   myapp:v3
```





