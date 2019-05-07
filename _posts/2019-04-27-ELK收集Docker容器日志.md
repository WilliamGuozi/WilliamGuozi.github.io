---
layout: post
title: "ELK收集Docker容器日志"
tags: DevOps ELK 
---

## 简介  

----



* 之前写过一篇博客 [ELK：日志收集分析平台](https://www.cnblogs.com/William-Guozi/p/elk.html)，介绍了在Centos7系统上部署配置使用ELK的方法，随着容器化时代的到来，容器化部署成为一种很方便的部署方式，收集容器日志也成为刚需。本篇文档从 **容器化部署ELK系统，收集容器日志，自动建立项目索引，ElastAlert日志监控报警，定时删除过期日志索引文件** 这几个方面来介绍ELK。
* 大部分配置方法多是看官方文档，理解很辛苦，查了很多文章，走了很多弯路，分享出来，希望让有此需求的朋友少走弯路，如有错误或理解不当的地方，请批评指正。
* 逻辑结构如下图

![img-w500](/images/201905051525.png)

&nbsp;
&nbsp;

## ELK之容器日志收集及索引建立  

----


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

&nbsp;
&nbsp;

## ELK之ElastAlert日志告警  

----


* ElastAlert分为两部分，后端程序，项目地址 <https://github.com/bitsensor/elastalert>，Kibana前端页面插件地址 <https://github.com/bitsensor/elastalert-kibana-plugin>
* ElastAlert目前支持的报警类型：
1. any：只要有匹配就报警；
1. blacklist：compare_key字段的内容匹配上 blacklist数组里任意内容；
1. whitelist：compare_key字段的内容一个都没能匹配上whitelist数组里内容；
1. change：在相同query_key条件下，compare_key字段的内容，在 timeframe范围内发送变化；
1. frequency：在相同 query_key条件下，timeframe 范围内有num_events个被过滤出来的异常；
1. spike：在相同query_key条件下，前后两个timeframe范围内数据量相差比例超过spike_height。其中可以通过spike_type设置具体涨跌方向是up,down,both 。还可以通过threshold_ref设置要求上一个周期数据量的下限，threshold_cur设置要求当前周期数据量的下限，如果数据量不到下限，也不触发；
1. flatline：timeframe 范围内，数据量小于threshold阈值；
1. new_term：fields字段新出现之前terms_window_size(默认30天)范围内最多的terms_size (默认50)个结果以外的数据；
1. cardinality：在相同 query_key条件下，timeframe范围内cardinality_field的值超过 max_cardinality 或者低于min_cardinality
1. Percentage Match: 在buffer_time 中匹配所设置的字段的百分比高于或低于阈值时，此规则将匹配。默认情况下为全局的buffer_time

### ElastAlert容器启动  
* `-p 3030:3030` 指定映射端口，该端口需要kibana调用
* `-v config/elastalert.yaml:/opt/elastalert/config.yaml` 映射ElastAlert的配置文件
* `-v rules:/opt/elastalert/rules` 映射报警规则文件目录
* `--link elasticsearch` ElastAlert需要连接ElasticSearch 进行数据的查询
* `rule_templates:/opt/elastalert/rule_templates` 该目录下为报警规则的模版，可使用其配置并放入 `rules` 使其生效，具体配置规则参考 `https://elastalert.readthedocs.io/en/latest/ruletypes.html`

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
* 可通过 `elastalert-test-rule rules/rule1.yaml` 在 `elastalert` 容器内测试报警规则
### Kibana 需要更改配置

* 添加 `elastalert` 相关的配置，包括主机名和端口

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

* 配置 Kibana的插件 `elastalert-kibana-plugin` 需要注意的是，插件的下载地址 <https://github.com/bitsensor/elastalert-kibana-plugin/releases> 需要找到对应自己 kibana 版本，否则插件无法安装成功
* 从目前版本来看，插件的使用不是很方便，其相当于在 ElasticAlert 启动时 `-v rules:/opt/elastalert/rules` 目录下写文件
* `-v /data/kibana/elastalert-kibana-plugin:/usr/share/kibana/plugins/elastalert-kibana-plugin` 映射插件目录

```bash
cd /data
wget https://github.com/bitsensor/elastalert-kibana-plugin/releases/download/1.0.2/elastalert-kibana-plugin-1.0.2-6.6.2.zip
unzip elastalert-kibana-plugin-1.0.2-6.6.2.zip
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
    -v /data/kibana/elastalert-kibana-plugin:/usr/share/kibana/plugins/elastalert-kibana-plugin \
    docker.elastic.co/kibana/kibana:6.6.2

```
* 启动后，可查看 `http://IP:5601`，可通过此来创建报警规则

![img-w500](/images/201905051152.png)

### 报警规则配置  

* 这里给出一个报警规则的示例，更多请查看文档 <https://elastalert.readthedocs.io/en/latest/ruletypes.html>
* `es_host: elasticsearch` 该配置会使用docker容器中的变量
* `name: filebeat info frequency rule` 报警规则的名称，需要唯一
* `timeframe:` 设定查询的时间尺度，这里指定为5分钟
* `filter` 指定报警条件，示例为日志中message字段还有INFO信息就示作匹配
* `alert` 指定报警方式，报警方式有很多，这里使用slack，可查看文档配置
 
```yaml
# Alert when the rate of events exceeds a threshold

# (Optional)
# Elasticsearch host
# es_host: elasticsearch.example.com
es_host: elasticsearch

# (Optional)
# Elasticsearch port
# es_port: 14900
es_port: 9200

# (OptionaL) Connect with SSL to Elasticsearch
#use_ssl: True

# (Optional) basic-auth username and password for Elasticsearch
#es_username: someusername
#es_password: somepassword

# (Required)
# Rule name, must be unique
name: filebeat info frequency rule

# (Required)
# Type of alert.
# the frequency rule type alerts when num_events events occur with timeframe time
type: frequency

# (Required)
# Index to search, wildcard supported
index: filebeat-*

# (Required, frequency specific)
# Alert when this many documents matching the query occur within a timeframe
num_events: 1

# (Required, frequency specific)
# num_events must occur within this amount of time to trigger an alert
timeframe:
#  hours: 1
  minutes: 5
# (Required)
# A list of Elasticsearch filters used for find events
# These filters are joined with AND and nested in a filtered query
# For more info: http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl.html

filter:
- query:
    query_string:
      query: "message:INFO"
# (Required)
# The alert is use when a match is found

alert:
- "slack"

alert_subject: "{} @{}"
alert_subject_args:
  - containername
  - log-timestamp
alert_text_type: alert_text_only
alert_text: |
  > 【Name】: {}
  > 【Log-Msg】: {}
alert_text_args:
  - containername
  - message


slack_webhook_url: "https://hooks.slack.com/services/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"
slack_title_link: "https://elk.glinux.top"
slack_title: "ELK URL"
slack_username_override: "ELK Alert"
slack_channel_override: "#filebeat-test"
```

* 报警的效果如图  

![img-w500](/images/201905051422.png)

&nbsp;
&nbsp;

## 定时删除过期日志索引文件  

----


* 该脚本用于删除7天以前的索引文件（过月日志清除），脚本为参考他人作品，做了少许更改，可查看参考文档
* `curl -s "http://127.0.0.1:9200/_cat/indices?v"` 该命令可以查看es中的索引
* `curl -XDELETE -s "http://127.0.0.1:9200/filebeat-2019.04.27"` 该命令用于删除es中的 `filebeat-2019.04.27` 索引

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
* 该脚本可以做成定时任务，每天都执行一次，来删除过期数据

&nbsp;
&nbsp;

## 参考文档

----


### ELK

1. 使用Docker搭建ELK日志系统   <https://zhuanlan.zhihu.com/p/32559371>
1. Beats详解（四) <http://www.51niux.com/?id=204>
1. Offical document: <https://www.elastic.co/guide/index.html>
1. 将日志输出到Docker容器外: <https://www.jianshu.com/p/bf2eb121ac62>
1. elk日志大盘显示和日志监控报警配置实践: <https://blog.csdn.net/yujiak/article/details/79897408>

### Elastalert 

1. elastalert github地址： <https://github.com/bitsensor/elastalert>
1. elastalert-kibana-plugin github地址: <https://github.com/bitsensor/elastalert-kibana-plugin/tree/1.0.2>
1. ElastAlert 官方操作文档：<https://elastalert.readthedocs.io/en/latest/ruletypes.html>

### 索引定期清除
1. 历史索引删除 <http://www.cenhq.com/2017/11/07/elasticsearch-deletes-index-by-date/>

### 其他
1. Centos 7 Docker命令自动补全 <https://medium.com/@ismailyenigul/enable-docker-command-line-auto-completion-in-bash-on-centos-ubuntu-5f1ac999a8a6>
1. Searchguard 插件支持OpenLdap：<https://github.com/floragunncom/search-guard-kibana-plugin>
1. 设置登录认证: <https://birdben.github.io/2017/02/08/Kibana/Kibana学习（六）Kibana设置登录认证>
1. SSL免费证书 <https://certbot.eff.org/lets-encrypt/ubuntuxenial-nginx>
