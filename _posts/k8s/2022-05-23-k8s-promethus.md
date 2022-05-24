---
layout: post
title: k8s生态中promethus的架构与原理研究
tags: [k8s, promethus]
lang: zh
---

# 架构
[https://prometheus.io/docs/introduction/faq/](https://prometheus.io/docs/introduction/faq/)

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/18490/1624887721086-a2f1f94c-2aae-4411-9115-6ec88216266e.png#align=left&display=inline&height=406&margin=%5Bobject%20Object%5D&name=image.png&originHeight=811&originWidth=1351&size=130294&status=done&style=none&width=675.5)

本质上是类似alimonitor的系统.
# 几个细节:

- 使用pull模式, 而不是push模式. 即promethus server主动去各个业务方捞取数据, 而不是各个业务方主动往promethus server推送数据.
    - 为啥使用pull模式? [https://prometheus.io/docs/introduction/faq/#why-do-you-pull-rather-than-push](https://prometheus.io/docs/introduction/faq/#why-do-you-pull-rather-than-push)
        - You can run your monitoring on your laptop when developing changes.
        - You can more easily tell if a target is down.
        - You can manually go to a target and inspect its health with a web browser.
    - 针对现存的已经有的push模式, 提供了pushgateway方案来兼容. (可以参见上图架构)
        - [https://prometheus.io/docs/instrumenting/pushing/](https://prometheus.io/docs/instrumenting/pushing/)
        - pushgateway类似于
- 由于使用了pull模式, 因此会有各种[exporter组件](https://prometheus.io/docs/instrumenting/exporters/), 来适配各个常用监控源, 例如 casandra exporter, ECS Exporter等. 本质上这种exporter是类似一个http服务器, 可以让promethus server来捞取数据.
- 服务发现: 由于使用的是pull模式, 因此promethus需要知道要从哪里捞取各个业务方的metrics, 因此需要一个服务发现机制. 有如下两种:
    - 静态文件配置(File-Based SD): ip:port
    - DNS: nameserver:port [https://prometheus.io/docs/prometheus/latest/configuration/configuration/#dns_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#dns_sd_config)
    - HTTP_SD:
- 存储:
    - 存储介质:
        - 默认使用的localstorage, 本地存储
        - 可以支持remote storage, 但由于各个remote storage的接口&对原生的promethus SQL支持度的不同, 分为read&write能力
            - [remote storage支持列表](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage)
    - 数据格式:
        - 自定义的, 支持压缩的本地存储格式.
- HA:
    - 服务HA:
        - [https://www.robustperception.io/scaling-and-federating-prometheus](https://www.robustperception.io/scaling-and-federating-prometheus)
        - 默认: 单台server足以支撑1000台server的metrics量.
        - 方案1: Initial Deployment
            - 2个global Prometheus , N个local Promethus, global负责汇聚local数据
            - 保持2个global, 防止单台宕机.
        - 方案2: Splitting By Use, 按用途垂直拆分
            - promethus前后端分离部署
            - 按业务域分别部署, 例如为Cassandra服务单独部署一套, 为MySQL服务单独部署一套
        - 方案3: Horizontal Sharding, 水平拆分
            - 不推荐多机器部署,
    - 数据HA:
        - localstorage: 无法HA, 因此promethus是推荐使用RAID, 或者定期打快照.

- 限制:
    - 不要用promethus来进行tracing/logging, 只能用来做metrics

# 核心术语/指标
定义了很好的几种通用指标, 来覆盖metrics各个方面:


1. counter: [https://www.robustperception.io/how-does-a-prometheus-counter-work](https://www.robustperception.io/how-does-a-prometheus-counter-work)
    1. rate
2. gauge
2. histogram, 如下, 按照le="xxx", le即"less than", 将latency分成了10个bucket
```
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.1"} 25547
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.2"} 26688
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.4"} 27760
prometheus_http_request_duration_seconds_bucket{handler="/",le="1"} 28641
prometheus_http_request_duration_seconds_bucket{handler="/",le="3"} 28782
prometheus_http_request_duration_seconds_bucket{handler="/",le="8"} 28844
prometheus_http_request_duration_seconds_bucket{handler="/",le="20"} 28855
prometheus_http_request_duration_seconds_bucket{handler="/",le="60"} 28860
prometheus_http_request_duration_seconds_bucket{handler="/",le="120"} 28860
prometheus_http_request_duration_seconds_bucket{handler="/",le="+Inf"} 28860
```

4. summary


# 核心组件研究
## pushgateway
[https://prometheus.io/docs/practices/pushing/](https://prometheus.io/docs/practices/pushing/)
[https://github.com/prometheus/pushgateway](https://github.com/prometheus/pushgateway)


## 存储
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/18490/1624937750293-3b18399a-dca5-489e-a79c-e57a679867c0.png#align=left&display=inline&height=96&margin=%5Bobject%20Object%5D&name=image.png&originHeight=96&originWidth=695&size=12233&status=done&style=none&width=695)



# 扩展
Observability的三大部分:

| Metrics | Promethus + Grafana(前端dashboard) |  |
| --- | --- | --- |
| Tracing | ELK |  |
| Logging |  |  |

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/18490/1624888180678-5feb535f-66dd-4e98-bebb-43694981f0c8.png#align=left&display=inline&height=256&margin=%5Bobject%20Object%5D&name=image.png&originHeight=400&originWidth=676&size=83578&status=done&style=none&width=432)
**

- Metrics比较好理解, 核心可以分为如下几层: [https://codeascraft.com/2011/02/15/measure-anything-measure-everything/](https://codeascraft.com/2011/02/15/measure-anything-measure-everything/) metrics are structured by default
    - network
    - machine: 现成的node exporter, [https://github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter)
        - disk/io
        - cpu/mem/threads/load...
        - status, ...
    - application (而通常应用的metrics是最为复杂的), 典型的应用, 已经有promethus的exporter了
        - DB
        -

- Logging:
- Tracing: 类似鹰眼, 根据traceId查看
  - 


# 一些有意思的思考
[https://www.splunk.com/en_us/data-insider/what-is-observability.html](https://www.splunk.com/en_us/data-insider/what-is-observability.html)

1. monitoring与observability的关系是啥?
    1. Monitoring is an action you perform to increase the observability of your system.
    1. Observability is a property of that system.
2. Apdex指标: [https://en.wikipedia.org/wiki/Apdex](https://en.wikipedia.org/wiki/Apdex)
2. EKL: Elasticsearch, Logstash, and Kibana