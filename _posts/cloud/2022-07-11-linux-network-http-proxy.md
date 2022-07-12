---
layout: post
title: Linux网络之HTTP Proxy原理与实现分析
subtitle: Linux网络之HTTP Proxy原理与实现分析
tags: [linux, network, cloud-computing, iaas, http-proxy, proxy]
---

# 背景
在使用apache的HttpClient时(获取其他常用的httpclient), 经常会发现初始化时可以配置http proxy. 但有些疑问:

1. 在什么场景下需要使用http proxy?
1. 具体http proxy怎么配置?
1. 具体http proxy的实现原理是啥? 包括 httpClient 如何使用这个proxy? 具体的httpProxy怎么处理这种包?

# 实践
## 环境

- docker
- ubuntu

## http proxy 服务端部署
Nginx通常是作为反向代理, 但其实也可以作为正向代理使用:  [nginx_proxy](https://github.com/reiz/nginx_proxy)

```shell
docker run -d -p 9999:8888 -v ${PWD}/nginx_allowlist.conf:/usr/local/nginx/conf/nginx.conf reiz/nginx_proxy:0.0.3
```

## http proxy 客户端配置
```java
import java.io.IOException;

import org.apache.commons.httpclient.HostConfiguration;
import org.apache.commons.httpclient.HttpClient;
import org.apache.commons.httpclient.methods.GetMethod;
import org.apache.commons.httpclient.params.HttpMethodParams;
import org.junit.Test;

public class HttpClientTest {
    private static final String ENCODING = "UTF-8";
    
    @Test
    public void name() throws IOException {
        String url = "https://www.google.com";
        
        HttpClient client = new HttpClient();
        HostConfiguration hostConfiguration = new HostConfiguration();
        hostConfiguration.setProxy("127.0.0.1", 9999);
        client.setHostConfiguration(hostConfiguration);
        client.getParams().setParameter(
            HttpMethodParams.HTTP_CONTENT_CHARSET, ENCODING);
        GetMethod method = new GetMethod(url);
        try {
            int i = client.executeMethod(method);
            System.out.println("Code: " + i);
        } catch (Exception e) {
            e.printStackTrace();
        }
        String response = method.getResponseBodyAsString();
        System.out.println(response);
    }
}
```

## 客户端请求分析
不配置proxy:

- client ---http get---> www.google.com

配置了proxy:

- client --- tcp ---> proxy --- http get --->  www.google.com

可以看到, client.executeMethod实质上是:

1. client先与proxy建立了TCP连接.
1. 然后将HTTP报文, 包括目标域名(即GET http://www.google.com), 作为TCP的output stream传给proxy
1. proxy解析tcp报文, 获取到真正的目标域名, 查找DNS获取到google.com的ip地址.
1. proxy与google.com建立TCP连接, 发送报文body等.

这种也叫: relay, 即中继.

## 实际连接分析

### host上， LISTEN了9999端口
```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/PlayGround/docker/nginx-forward-proxy$ netstat -natp | fgrep 9999
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:9999            0.0.0.0:*               LISTEN      -                   
```

### 容器内部， LISTEN了8888端口
```shell
root@84129a407795:/app# netstat -nat | fgrep 8888
tcp        0      0 0.0.0.0:8888            0.0.0.0:*               LISTEN     
```

### 在Host上某个java进程发起一个HTTP请求之后

- HOST上 Java 进程
```shell
davywalker@davywalker-ThinkPad-X1-Carbon-4th:~/Downloads/Clash-Linux$ netstat -natp | fgrep 9999
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:9999            0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:9999          127.0.0.1:42261         TIME_WAIT   -                   
```

- 容器内部
```shell
root@84129a407795:/app# netstat -nat | fgrep 8888
tcp        0      0 0.0.0.0:8888            0.0.0.0:*               LISTEN     
tcp        0      0 172.17.0.4:8888         172.17.0.1:51960        TIME_WAIT  
```

TODO:
- 在访问`http://www.google.com`时，Java测试代码正常返回结果。
- 在访问`http://www.baidu.com`时， Java会出现如下异常，理论上由于不在nginx的白名单里应该被block的，但为啥会抛出异常？
```java
java.io.IOException: Stream closed
	at java.io.BufferedInputStream.getBufIfOpen(BufferedInputStream.java:170)
	at java.io.BufferedInputStream.read(BufferedInputStream.java:336)
	at org.apache.commons.httpclient.WireLogInputStream.read(WireLogInputStream.java:69)
	at org.apache.commons.httpclient.ContentLengthInputStream.read(ContentLengthInputStream.java:170)
	at java.io.FilterInputStream.read(FilterInputStream.java:133)
	at org.apache.commons.httpclient.AutoCloseInputStream.read(AutoCloseInputStream.java:108)
	at java.io.FilterInputStream.read(FilterInputStream.java:107)
	at org.apache.commons.httpclient.AutoCloseInputStream.read(AutoCloseInputStream.java:127)
	at org.apache.commons.httpclient.HttpMethodBase.getResponseBody(HttpMethodBase.java:690)
	at org.apache.commons.httpclient.HttpMethodBase.getResponseBodyAsString(HttpMethodBase.java:803)
	at edu.xmu.test.javaweb.httpclient.HttpClientProxyTest.name(HttpClientProxyTest.java:31)
```



# 原理




# 再回到实践中

## 实践1： DNS劫持+正向代理过滤域名
通过DNS劫持，将所有HTTP请求都定位到正向代理中，在正向代理里进行代理域名的黑白名单控制。

- 缺点：
  - DNS劫持可以被手动禁用，也可以手动指定`8.8.8.8`绕过DNS服务器（当然如果DNS请求直接都被劫持，）。
  - 只针对HTTP请求有效，如果知道目标的IP, 可以通过IP直连上。





