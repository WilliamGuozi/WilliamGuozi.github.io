---
layout: post
title: "ELK收集Docker容器日志"
tags: DevOps ELK 
---

## 简介
* 之前写过一篇博客 [ELK：日志收集分析平台](https://www.cnblogs.com/William-Guozi/p/elk.html)，介绍了在Centos7系统上部署配置使用ELK的方法，随着容器化时代的到来，容器化部署成为一种很方便的部署方式，收集容器日志也成为刚需。本篇文档从 **容器化部署ELK系统，收集容器日志，自动建立项目索引，ElastAlert日志监控报警，定时删除过期日志索引文件** 这几个方面来介绍ELK。
* 大部分配置方法多是看官方文档，理解很辛苦，查了很多文章，走了很多弯路，分享出来，希望让有此需求的朋友少走弯路，如有错误或理解不当的地方，请批评指正。

## ELK之容器日志收集及索引建立
* Docker环境下，拉取官方镜像，官方镜像地址 [Docker @ Elastic](https://www.docker.elastic.co)
```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:6.6.2
docker pull docker.elastic.co/kibana/kibana:6.6.2
docker pull docker.elastic.co/beats/filebeat:6.6.2
docker pull docker.elastic.co/logstash/logstash:6.6.2
```


### Elasticsearch 容器部署  

> Elasticsearch 启动命令说明
* 启动两个端口，9200为数据查询和写入端口，也即是业务端口，9300为集群端口，同步集群数据，此处我单节点部署
* 指定日志输出格式为json格式
* 日志文件最多保留三个，每个最多10M
* 容器开机自启
* 传递参数为单节点部署
* 数据存储映射至宿主机
* 需要给 `/data/elasticsearch` 赋予权限，否则报权限不足的错误  

```bash
docker run -d \
    --user root \
    -p 127.0.0.1:9200:9200 \
    -p 9300:9300 \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    --name elasticsearch \
    --restart=always \
    -e "discovery.type=single-node" \
    -v /data/elasticsearch:/usr/share/elasticsearch/data \
    docker.elastic.co/elasticsearch/elasticsearch:6.6.2
    
chmod 777 -R /data/elasticsearch
```
### Logstash 容器部署  

> Logstash 配置文件说明  

* 配置文件映射至至宿主机
* `match => { "message" => "%{TIMESTAMP_ISO8601:log-timestamp} %{LOGLEVEL:log-level} %{JAVALOGMESSAGE:log-msg}" }
` 将日志做简单拆分, 时间戳重命名为 `log-timestamp`，日志级别重命名为 `log-msg`
* 通过 `remove_field => ["beat"]` 移除无用字段
* 输出到elasticsearch
* 索引通过 容器名称-时间 建立  

```yaml
mkdir -pv /data/conf
cat > /data/conf/logstash.conf << "EOF"
input {
  beats {
    host => "0.0.0.0"
    port => "5043"
  }
}
filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:log-timestamp} %{LOGLEVEL:log-level} %{JAVALOGMESSAGE:log-msg}" }
  }
  mutate {
#    remove_field => ["message"]
    remove_field => ["beat"]
   }
}
output {
  stdout { codec => rubydebug }
  elasticsearch {
        hosts => [ "elasticsearch:9200" ]
        index => "%{containername}-%{+YYYY.MM.dd}"
  }
}
EOF
```  

> Logstash 启动命令说明  
* 通过 `--link elasticsearch` 连接 `elasticsearch` 容器，并生成环境变量在容器中使用
* 配置文件映射如容器  

```bash
docker run -p 5043:5043 -d \
    --user root \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    --name logstash \
    --link elasticsearch \
    --restart=always \
    -v /data/conf/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
    docker.elastic.co/logstash/logstash:6.6.2
```
### Kibana容器部署  

* 启动参数参考上述组件
```bash
docker run -p 5601:5601 -d \
    --user root \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    --name kibana \
    --link elasticsearch \
    --restart=always \
    -e ELASTICSEARCH_URL=http://elasticsearch:9200 \
    docker.elastic.co/kibana/kibana:6.6.2
```
* 通过服务器IP地址即可访问Kibana web `http://IP:5601`
### Filebeat 容器部署  

> Filebeat 配置文件说明
* 
```yaml
cat > /data/conf/filebeat.yml << "EOF"
filebeat.inputs:
- type: log
  paths:
   - '/var/lib/docker/containers/*/*.log'
  json.add_error_key: true
  json.overwrite_keys: true
  json.message_key: log
  multiline:
      pattern: ^\d{4}
      negate: true
      match: after
# 排除或删除通配符行
  exclude_lines: ['^DBG']
  include_lines: ['ERROR', 'WARN', 'INFO']
# 排除通配文件
#  exclude_files: ['\.gz$']
  processors:
    - add_docker_metadata:
        match_source: true
#选取添加docker元数据的层级
        match_source_index: 4
processors:
- rename:
    fields:
     - from: "json.log"
       to: "message"
     - from: "docker.container.name"
       to: "containername"
     - from: "log.file.path"
       to: "filepath"
- drop_fields:
    fields: ["metadata","beat","input","prospector","host","source","offset"]
output.logstash: # 输出地址
  hosts: ["192.168.1.42:5043"]
#output.elasticsearch:
#  hosts: ["172.16.131.35:19200"]
#  protocol: "http"
#output.file:
#  path: "/data/logs"
#  filename: filebeat.out
#output.console:
#  pretty: true
EOF
```  

> Filebeat 启动命令
```bash
docker run -d \
    --name filebeat \
    --user root \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    --restart=always \
    -v /data/conf/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro \
    -v /var/lib/docker/containers/:/var/lib/docker/containers/ \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    docker.elastic.co/beats/filebeat:6.6.2

```

* Filebeat是日志收集客户端，正常情况下其部署在需要收集日志的主机上

## ELK之ElastAlert日志告警
## 定时删除过期日志索引文件