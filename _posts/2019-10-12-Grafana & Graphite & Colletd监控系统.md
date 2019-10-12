---
layout: post
title: "Grafana & Graphite & Collectd监控系统"
tags: DevOps Monitor
---


## 简介  
----
* 监控是运维工作中的一个重要组成部分，今天介绍一套新的监控工具，方便好用，扩展性强，这套工具有三个组件，Grafana & Graphite & Collectd；
* [Grafana](https://grafana.com/) 是一个开源的强有力的数据展示、量化分析工具，数据源包括 graphite、prometheus、mysql、influxdb 等等，可以直接在页面上组装语句，另外还可以对资源实现可用性和性能监控报警，同时还支持集成OpenLDAP；
* [Graphite](https://graphiteapp.org) 是一个用Python写的开源的监控绘图工具，由三个组建组成，分别是 `carbon（基于 Twisted 的进程，用来接收数据）`、`whisper（专门存储时间序列类型数据的小型数据库）`、`graphite-web（基于 Django 的网页应用程序）`，我们这里使用其存储监控数据；
* [Collectd](https://collectd.org/) 是一个用C语言开发的守护进程，能够周期性的收集系统和应用程序的性能指标，同时给各种存储方式提供不同的存储机制，我们这里使用其收集数据并将数据推送到 graphite 中存储；
* 目前云平台使用普遍且方便，大多数云平台有完善详尽的监控预警系统，但对于业务需要使用多种云平台或混合云的情况却较难应对；这套监控体系可综合各种系统监控、业务监控、业务数据展示等功能，统一入口，可谓运维必备利器；现将该系统的创建分享方向给大家，希望对需要的小伙伴有所帮助；
* 下图为grafana 可以接收的数据源列表  
![img-w500](/images/201909241731.png)

## graphite 部署
----
* 数据做持久存储
```bash
docker run -d \
 --name ops-graphite \
 --restart=always \
 -p 8880:80 \
 -p 2003-2004:2003-2004 \
 -p 2023-2024:2023-2024 \
 -p 8125:8125/udp \
 -p 8126:8126 \
 -v /opt/graphite_data/whisper:/opt/graphite/storage/whisper:rw \
 -v /opt/graphite_data/redis:/var/lib/redis:rw \
 -v /opt/graphite_data/log:/var/log:rw \
 graphiteapp/graphite-statsd
```
* 可通过浏览器访问 graphite 页面，`http://10.0.0.1:8880`，默认用户名：root，密码：root，后续要将其加入到grafana的数据源

## collectd 部署
----
* 替换 `GRAPHITE_HOST` 为你graphite的主机地址，我这里使用域名，方便管理
```bash
docker run -d \
 --name ops-collectd \
 --net=host \
 --privileged \
 --restart always \
 -v /:/hostfs:ro \
 -e GRAPHITE_HOST=collectd.ops.glinux.top \
 williamguozi/collectd:latest
```

## grafana 部署
----
* 数据做持久存储，可通过 `-v /opt/grafana/grafana.ini:/etc/grafana/grafana.ini`， ` -v /opt/grafana/ldap.toml:/etc/grafana/ldap.toml` 将配置放置外部管理（可选）
```bash
docker run -d \
  --name ops-grafana \
  -p 3000:3000 \
  -v /opt/grafana:/var/lib/grafana \
  grafana/grafana
```

## grafana 配置
----
* 经过上诉配置，就可以打开grafana的管理界面了，`http://10.0.0.1:3000`，默认用户名：admin，密码：admin
* 添加 `graphite` 数据源，配置用户名密码，测试连接状态  
![img-w500](/images/201910121605.png)
* 设置告警通知方式，这里使用slack方式通知到频道，也可尝试其他通知方式  
![img-w500](/images/201910121711.png)
* 左侧列表添加 Dashboard -> Panel，编辑Panel，添加数据，比如cpu利用率  
![img-w500](/images/201910121625.png)
* 调整单位  
![img-w500](/images/201910121631.png)
* 修改 Panel 名称，添加报警规则  
![img-w500](/images/201910121735.png)


## 效果展示
----
* 当资源指标达到阈值就会报警到Slack相应的频道  
![img-w500](/images/201910121741.png)
* 另外，可以通过安装 [dashboards](https://grafana.com/grafana/dashboards) 模版使数据展示更漂亮  
![img-w500](/images/201910121750.png)

## 总结
----
* 本文主要就操作系统的基础监控做例子，展示整个部署过程及展示和报警；
* 当然其也能够对时下比较流行的kubernetes进行详细的监控，后面会写文介绍；
* 另外，graphite可以直接将mysql作为数据源，将业务数据图标展示，体现DevOps价值；

## 参考文档
----
* Installing Graphite: <https://graphite.readthedocs.io/en/latest/install.html>
* Graphite简介: <https://my.oschina.net/u/1263964/blog/701664>
* Graphite 和 Grafana 简介: <https://yumminhuang.github.io/post/graphiteandgrafana>
* 创建一个slack api: <https://api.slack.com/start>