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

#### Elasticsearch 启动命令说明  

* 启动两个端口，9200为数据查询和写入端口，也即是业务端口，9300为集群端口，同步集群数据，此处我单节点部署
* 指定日志输出格式为json格式，默认情况下也为json格式，默认输出路径 `/var/lib/docker/containers/*/*.log`
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

#### Logstash 配置文件说明  

* 配置文件映射至至宿主机
* `match => { "message" => "%{TIMESTAMP_ISO8601:log-timestamp} %{LOGLEVEL:log-level} %{JAVALOGMESSAGE:log-msg}" }
` 将日志做简单拆分, 时间戳重命名为 `log-timestamp`，日志级别重命名为 `log-msg`，如果拆分成功输出日志中就会产生这两个字段
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

#### Logstash 启动命令说明  

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

#### Kibana配置文件说明

```yaml

# Default Kibana configuration from kibana-docker.

server.name: kibana
server.host: "0"
elasticsearch.hosts: http://elasticsearch:9200
xpack.monitoring.ui.container.elasticsearch.enabled: true
elastalert-kibana-plugin.serverHost: 192.168.30.42
elastalert-kibana-plugin.serverPort: 3030
```

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
    -v /data/conf/kibana.yml:/usr/share/kibana/config/kibana.yml \
    -v /data/elastalert-kibana-plugin:/usr/share/kibana/plugins/elastalert-kibana-plugin \
    docker.elastic.co/kibana/kibana:6.6.2
```
* 通过服务器IP地址即可访问Kibana web `http://IP:5601`

### Filebeat 容器部署  

#### Filebeat 配置文件说明  

* Filebeat是一个轻量级的日志收集工具，是Elastic的Beat组成员，默认情况下，docker输出的日志格式为json格式，需要解码才方便查看日志的路径为 `/var/lib/docker/containers/*/*.log`
* `- type: log` 选择filebeat输入类型选择log，之后通过解码json的配置将docker输出的日志解码，输入类型也可选择docker，但是我试过目前版本该数据类型有些功能受限，比如后面rename时，只能rename一个
* json解码： 必须至少指定设置其中之一来启用JSON解析模式，建议三行都加上，不然，有些功能会有问题
* json.overwrite_keys   #如果启用了keys_under_root和此设置，那么来自解码的JSON对象的值将覆盖Filebeat通常添加的字段（type, source, offset,等）以防冲突。
* json.add_error_key    #如果启用此设置，则在出现JSON解组错误或配置中定义了message_key但无法使用的情况下，Filebeat将添加“error.message”和“error.type：json”键。
* json.message_key      #一个可选的配置设置，用于指定应用行筛选和多行设置的JSON密钥。 如果指定，键必须位于JSON对象的顶层，且与键关联的值必须是字符串，否则不会发生过滤或多行聚合。
* 多行合并： 将多行日志合并成一条，比如java的报错日志，规则为，如果不是以四个数字，即年份开头的日志合并到上一条日志当中
* 排除或删除通配符行： 即排除含有相关关键字的行，或只收集含有相关关键字的行
* 排除通配文件： 即不读取该含该字符的文件的日志
* 添加docker元数据： 将docker容器的相关数据收集起来，方便作为索引的字段， `match_source_index: 4` 为获取docker数据的索引，其数据为json格式，索引太低可能无法获取关键docker数据
* 处理器重命名和删除无用字段： `- rename:` 将上面收集的元数据重命名来提升其索引等级，以便交给logstash来处理，其中 `docker.container.name` 为docker容器的名字，重命名为 `containername`字段， `- drop_fields:` 清除掉无用索引数据，不输出到 output
* 日志输出：日志输出的几种方式 `output.file` 输出到文件，`output.console` 输出到标准输出，通过 `docker logs containName` 查看，该两种输出方便调试  

```yaml
cat > /data/conf/filebeat.yml << "EOF"
filebeat.inputs:
- type: log
  paths:
   - '/var/lib/docker/containers/*/*.log'
# json解码
  json.add_error_key: true
  json.overwrite_keys: true
  json.message_key: log
# 多行合并
#  multiline:
#    pattern: ^\d{4}
#    negate: true
#    match: after
#
# 排除或删除通配符行
#  exclude_lines: ['^DBG']
#  include_lines: ['ERROR', 'WARN', 'INFO']
# 排除通配文件
#  exclude_files: ['\.gz$']
#
# 添加docker元数据
  processors:
    - add_docker_metadata:
        match_source: true
# 选取添加docker元数据的层级
        match_source_index: 4
# 处理器重命名和删除无用字段
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
    fields: ["docker","metadata","beat","input","prospector","host","source","offset"]
# 日志输出
output.logstash: # 输出地址
  hosts: ["192.168.30.42:5043"]
#output.elasticsearch:
#  hosts: ["192.168.30.42:9200"]
#  protocol: "http"
#output.file:
#  path: "/tmp"
#  filename: filebeat.out
#output.console:
#  pretty: true
EOF
```  

#### Filebeat 启动命令 

* `-v /var/lib/docker/containers/:/var/lib/docker/containers/` 映射日志目录
* `-v /var/run/docker.sock:/var/run/docker.sock:ro` 映射docker套接字文件，来收集docker信息，添加docker元数据  

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

### 添加kibana索引
* 通过服务器IP地址访问Kibana web `http://IP:5601` 同时添加索引  
![img-w500](/images/201905011954.png)

* 如此便可查看相关日志  
![img-w500](/images/201905011747.png)

## ELK之ElastAlert日志告警 

### ElastAlert容器启动 

```bash
git clone https://github.com/bitsensor/elastalert.git; cd elastalert
sed -i -e 's/localhost/elasticsearch/g' `pwd`/config/elastalert.yaml
docker run -d \
    -p 3030:3030 \
    -v `pwd`/config/elastalert.yaml:/opt/elastalert/config.yaml \
    -v `pwd`/config/config.json:/opt/elastalert-server/config/config.json \
    -v `pwd`/rules:/opt/elastalert/rules \
    -v `pwd`/rule_templates:/opt/elastalert/rule_templates \
    --link elasticsearch \
    --name elastalert \
    bitsensor/elastalert:latest
```

### Kibana 需要更改配置

```yaml
cat > /data/conf/kibana.yml << "EOF"
# Default Kibana configuration from kibana-docker.

server.name: kibana
server.host: "0"
elasticsearch.hosts: http://elasticsearch:9200
xpack.monitoring.ui.container.elasticsearch.enabled: true
elastalert-kibana-plugin.serverHost: elastalert
elastalert-kibana-plugin.serverPort: 3030
```

### Kibana 启动参数更改

```bash
docker run -p 5601:5601 -d \
    --user root \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    --name kibana \
    --link elasticsearch \
    --link elastalert \
    --restart=always \
    -e ELASTICSEARCH_URL=http://elasticsearch:9200 \
    -v /data/conf/kibana.yml:/usr/share/kibana/config/kibana.yml \
    -v /data/elastalert-kibana-plugin:/usr/share/kibana/plugins/elastalert-kibana-plugin \
    docker.elastic.co/kibana/kibana:6.6.2

docker exec -it kibana bash
kibana-plugin install https://github.com/bitsensor/elastalert-kibana-plugin/releases/download/1.0.2/elastalert-kibana-plugin-1.0.2-6.6.2.zip
echo "elastalert-kibana-plugin.serverHost: elastalert" >> config/kibana.yml
echo "elastalert-kibana-plugin.serverPort: 3030">> config/kibana.yml
docker restart kibana ;docker logs kibana -f
```


## 定时删除过期日志索引文件  

```bash
cat > /data/delete_index.sh << "EOF"
#!/bin/bash

elastic_url=127.0.0.1
elastic_port=9200
user="username"
pass="password"
log="/data/log/delete_index.log"


# 当前日期时间戳减去索引名时间转化时间戳是否大于1
dateDiff (){
    dte1=$1
    dte2=$2
    diffSec=$((dte1-dte2))
    if ((diffSec > 6)); then
	echo "1"
    else
        echo "0"
    fi
}


# 循环获取索引文件，通过正则匹配过滤
for index in $(curl -s "${elastic_url}:${elastic_port}/_cat/indices?v" | awk -F' ' '{print $3}' | grep -E "[0-9]{4}.[0-9]{2}.[0-9]{2}$"); do
# 获取索引文件日期，并转化格式
  date=$(echo $index | awk -F'-' '{print $NF}' | sed -n 's/\.//p' | sed -n 's/\.//p')
# 获取当前日期
  cond=$(date '+%Y%m%d')
# 根据不同服务器，计算不同数值
  diff=$(dateDiff "${cond}" "${date}")
# 打印索引名和结果数值
  #echo -n "${index} (${diff})"

# 判断结果值是否大于等于1
  if [ $diff -eq 1 ]; then
    curl -XDELETE -s "${elastic_url}:${elastic_port}/${index}" && echo "${index} 删除成功" >> $log || echo "${index} 删除失败" >> $log
fi
done
EOF

```

## 参考文档
### ELK

1. 使用Docker搭建ELK日志系统  https://zhuanlan.zhihu.com/p/32559371
1. Beats详解（四) http://www.51niux.com/?id=204
1. 设置登录认证: https://birdben.github.io/2017/02/08/Kibana/Kibana学习（六）Kibana设置登录认证
1. CentOS7 FileBeat 6.2.2 简单记录: https://www.jianshu.com/p/b7245ce58c6a
1. Offical document: https://www.elastic.co/guide/index.html
1.  将日志输出到Docker容器外: https://www.jianshu.com/p/bf2eb121ac62
1. SSL免费证书：https://certbot.eff.org/lets-encrypt/ubuntuxenial-nginx

### Elastalert 

1. elk日志大盘显示和日志监控报警配置实践: https://blog.csdn.net/yujiak/article/details/79897408
1. ElastAlert: https://github.com/Yelp/elastalert https://elastalert.readthedocs.io/en/latest/
1. ElastAlert 基于Elasticsearch的监控告警: https://anjia0532.github.io/2017/02/14/elasticsearch-elastalert
1. ElastAlert: https://blog.xizhibei.me/2017/11/19/alerting-with-elastalert

1. Centos 7 Docker命令自动补全：https://medium.com/@ismailyenigul/enable-docker-command-line-auto-completion-in-bash-on-centos-ubuntu-5f1ac999a8a6

### 索引定期清除
1. 历史索引删除：http://www.cenhq.com/2017/11/07/elasticsearch-deletes-index-by-date/