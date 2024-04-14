---
title: ELK-docker 实践
author: 李岩
tags:
  - elk
categories:
  - 程序人生
  - ''
date: 2022-01-04 00:06:00
---
# ELK docker 部署实践
> 本文主要对 ELK 套件中的 filebeat, logstash, elasticsearch 的安装进行实践，以及简单运行。

<!-- more -->

# Elasticsearch 安装
这里参照官网给出的`docker-compose.yml`文件设置`elasticsearch`集群。`elastisearch`支持`single-node`及`multi node cluster`两种部署模式。在本文中，实际上两种方式都能达到效果。使用`single-node`启动的环境，查看集群状态时会出现`status:yellow`。将`docker-compose.yml`文件放置在一个单独的目录下，然后创建`data01, data02, data03`目录。依据实际需要，还可创建`plugins`目录映射。

```
version: '2'
services:
  es01:
    container_name: es01
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    ports:
      - 9200:9200
      - 9300:9300
    volumes:
      - ./data01:/usr/share/elasticsearch/data
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02
      - cluster.initial_master_nodes=es01,es02
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms128m -Xmx128m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elastic

  es02:
    container_name: es02
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01
      - cluster.initial_master_nodes=es01,es02
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms128m -Xmx128m"
    volumes:
      - ./data02:/usr/share/elasticsearch/data
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elastic

  es03:
    container_name: es03
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms128m -Xmx128m"
    volumes:
      - data03:/usr/share/elasticsearch/data
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elastic

volumes:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local

networks:
  elastic:
    driver: bridge
    external: true
```

注意这里将集群的网络设置为`external`，这样后续的`logstash`才能找到服务节点。此外，由于笔者的机器可用存储较小，因此设置`es`的存储占用设置为`128m`。实际使用时，可以按照需求进行调整。
运行`docker-compose up -d`即可后台启动。启动后，`curl -X GET "localhost:9200/_cat/nodes?v=true&pretty"` 判断集群状态。    

# Logstash 安装  
> collect, parse transform logs  

```
version: '2'
services:
  logstash:
    image: docker.elastic.co/logstash/logstash:7.15.0
    container_name: logstash
    user: root
    ports:
      - 5004:5004
    volumes:
      - ./config:/usr/share/logstash/config/
      - /etc/localtime:/etc/localtime
    command: bash -c "cd /usr/share/logstash && logstash -f config/online.conf"
    networks:
      - elastic

networks:
  elastic:
    driver: bridge
    external: true
```

```
# ./config/online.conf
input {
  # 这里支持多种 input
  beats {
    port => 5004
    codec => "json"
  }
}

filter {
  # 这里基于 ruby 脚本进行过滤
  ruby {
    path => "./config/filter.rb"
  }
}

output {
  # 这里将过滤后的结果输出到标准输出及 es 中
  stdout {
    codec => json
  }
  elasticsearch {
    hosts => ["es01:9200"]
    index => "logstash"
    #user => ""
    #password => ""
  }
}
```

```
# config/filter.rb
# 按照 online.conf 中的配置，logstash 启动后，会读取 filter.rb，并使用 filter 函数作为过滤函数。
require "json"

BEGIN{
  puts "start event filter"
}

END{
  puts "end event filter"
}

def filter(event)
  puts event
  if event.get("[errno]") != 0
    return []
  end
  valid_age = 0
  event.get("[data]").each{ | info |
    if info["age"] < 10
      valid_age += info["age"]
    end
  }
  event.set("[data]", valid_age)
  return [event]
end

```

logstash 启动后，会监听容器内的 `5004` 接口（配置于`online.conf`中），如果有信息传入，则会经过`filter.rb`中的 `filter()` 函数对数据进行处理。而后输出到标准输出及 `es01`容器`5004`端口的`elasticsearch`服务。由于`elasticsearch`及`logstash`容器使用了相同的网络，因此可以互相感知。  
# filebeat 安装
> filebeat 作为轻量级的日志收集器，仅占用很少的资源，即可完成日志的采集，并且转发至配置的`logstash`进行后续的处理、归档等操作。  

```
version: '2'
services:
  filebeat:
    image: docker.elastic.co/beats/filebeat:7.16.0
    container_name: filebeat
    user: root
    environment:
      - strict.perms=false
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml
      - ./data:/usr/share/filebat/data
    networks:
      - elastic
    command: bash -c "cd /usr/share/filebeat && filebeat -e -c ./filebeat.yml"

networks:
  elastic:
    driver: bridge
    external: true
```

```
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /usr/share/filebeat/input.log

filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

filbeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

output.logstash:
  hosts: "logstash:5004"
```


容器启动后，会监听`/usr/share/filebeat/input.log`文件。当该文件发生变更时，`filebeat`会读取增量的内容并进行转发。

# let it run
经过上述步骤，一个简单的日志监听、采集、处理、存档的流程就构建了。为了测试，可以在`filebeat`容器的`/usr/share/filebeat/input.log`中写入：

```
{"errno": 0,"data": [{"age": 9,"name": "tt"},{"age": 8,"name": "gg"}]}
```  

按照`logstash:online.conf`的逻辑，会向`elasticsearch`的`logstash`写入信息。
# 参考文献
1. [Linux-ELK日志收集](https://www.cnblogs.com/zhangfushuai/p/14975307.html)
2. [Install Elasticsearch with Docker](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker)
3. [Logstash介绍](https://www.cnblogs.com/cjsblog/p/9459781.html)
