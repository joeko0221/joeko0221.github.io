---
layout: post
author: Joe Ko
title: Elastic Stack + Filebeat on Docker (ILM 版本)
date: 2021-08-20 14:20 +0800
categories:
- elastic
tags:
- docker
toc:  true
---

網路上有非常多 elastic stack 相關文章，本文特點如下:

- 使用 docker 一鍵安裝 elastic stack
- 使用 filebeat 推送 log
- 使用 Index Lifecycle Management 管理 index

## 使用 Docker 一鍵安裝 elastic stack

[完整代碼下載](https://github.com/joeko0221/elastic-stack-filebeat-ilm-on-docker)

{% highlight yaml linenos %}
version: '2.2'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.13.4
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.13.4
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data02:/usr/share/elasticsearch/data
    networks:
      - elastic

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.13.4
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data03:/usr/share/elasticsearch/data
    networks:
      - elastic

  kib01:
    image: docker.elastic.co/kibana/kibana:7.13.4
    container_name: kib01
    ports:
      - 5601:5601
    environment:
      ELASTICSEARCH_URL: http://es01:9200
      ELASTICSEARCH_HOSTS: '["http://es01:9200","http://es02:9200","http://es03:9200"]'
    networks:
      - elastic

  logstash01:
    image: docker.elastic.co/logstash/logstash:7.13.4
    container_name: logstash01
    volumes:
      - /home/joe/elastic-stack-filebeat-ilm-on-docker/logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
      - /home/joe/elastic-stack-filebeat-ilm-on-docker/logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - 5044:5044
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elastic
    depends_on:
      - es01
      - es02
      - es03

  filebeat01:
    image: docker.elastic.co/beats/filebeat:7.13.4
    container_name: filebeat01
    entrypoint: "filebeat -e -strict.perms=false"
    volumes:
      - /home/joe/elastic-stack-filebeat-ilm-on-docker/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml
      - /home/joe/elastic-stack-filebeat-ilm-on-docker/logs:/home/filebeat/logs
    networks:
      - elastic
    depends_on:
      - es01
      - es02
      - es03
      - logstash01
  
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
 
{% endhighlight %}

- elasticsearch, kibana 的設定，直接複製[官網](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html)的範例即可
- logstash 需掛載 volumes，覆蓋 logstash.yml, pipeline 目錄
- filebeat 須掛載 volumes，覆蓋 filebeat.yml, logs 目錄
- 最後 $ docker-compose up 一鍵安裝

### logstash 設定 
[代碼下載](https://github.com/joeko0221/elastic-stack-filebeat-ilm-on-docker/blob/main/logstash/pipeline/logstash.conf)
{% highlight config linenos %}
input {
  beats {
    port => 5044
  }
}

filter {
  grok {    
    match => [ "message", "%{TIMESTAMP_ISO8601:logTimestamp} %{DATA:logType} %{DATA:threadNm} %{DATA:logger} - %{GREEDYDATA:detail}" ]
  }
}

output {
  elasticsearch {
    hosts => "http://es01:9200"
    #index => "elastic-stack-filebeat-ilm-on-docker_index"
    ilm_rollover_alias => "elastic-stack-filebeat-ilm-on-docker"
    ilm_pattern => "000001"
    ilm_policy => "elastic-stack-filebeat-ilm-on-docker_policy"
  }
}
{% endhighlight %}

- 先暫時指定 index 名稱 (index => "elastic-stack-filebeat-ilm-on-docker_index")
- elastic stack 啟動完成，並設定 ILM policy 後，再修改為 ILM 方式


### filebeat 設定
[代碼下載](https://github.com/joeko0221/elastic-stack-filebeat-ilm-on-docker/blob/main/filebeat/filebeat.yml)
{% highlight yaml linenos %}
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /home/filebeat/logs/*.log

output.logstash:
  hosts: ["logstash01:5044"]

{% endhighlight %}


## 設定 ILM policy
![placeholder](https://joeko0221.github.io/images/ilm-policy-setting.png "ilm policy 設定")
- kibana 左方選單 -> Stack Management -> Index Lifecycle Policies -> Create policy
- 設定完成後，Linked Indices 數字為 1，就代表 policy 和 index 綁定完成囉
- 每五天自動更換 index，並刪除過期 index

-----
