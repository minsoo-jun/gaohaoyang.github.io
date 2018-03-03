---
layout: post
title:  "elasticdump"
date:   2018-02-28 15:14:54
categories: elasticsearch
tags: elasticsearch
excerpt: elasticdump를 이용해서 Elasticsearch index data를 퍼나르자.
mathjax: true
---

Elasticsearch > elasticdump
===================
Created by Minsoo jun, last modified on 2017-07-19

#[elasticdump](https://www.npmjs.com/package/elasticdump)

아파치 로그를 저장하고 있던 Elasticsearch서버를 새로운 클러스터로 이전할 필요가 있어서 어떻게 할까 고민하다가 두가지 안이 생각났습니다.
* 로그스타쉬([Logstash](https://www.elastic.co/products/logstash))을 이용해서 하는 방법
    * 로그스타쉬의 [input](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-elasticsearch.html)을 원본 elasticsearch로 잡고 [filter](https://www.elastic.co/guide/en/logstash/6.2/filter-plugins.html)에서 필요한 처리를 한후 [output](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html)으로 새로운 클러스터로 보내기

![logstash](https://www.elastic.co/assets/blt8a9ac25aedbd9ca7/logstash-img1.png)

* elasticdump을 이용해서 인덱스별로 이전을 하자

![elasticdump](https://raw.github.com/taskrabbit/elasticsearch-dump/master/elasticdump.jpg)

원타임 작업이라서 logstash을 위한 새로운 환경을 구축하기보다는 갱신이 없는 인덱스에 대해서 elasticdump로 인덱스 단위로 이전을 하고, 현재 갱신중인 인덱스에 대해서는 logstash의 output을 신/구 클러스터의 양쪽으로 보내는 것으로 했습니다.

[elasticdump](https://www.npmjs.com/package/elasticdump)홈페이지를 확인 하시면 자세한 사용법이 있습니다 꼭 참조 하세요. elasticsearch에서 elasticsearch로 보내는 것 이외에도 파일로 출력하는 등의 기능도 있습니다.

elasticdump를 실행하기 위해서는 nodejs가 필요합니다.

Install
=======

Install nvm
```
 cd /usr/local/
 git clone https://github.com/creationix/nvm.git
 
 chmod +x /usr/local/nvm/nvm.sh
 /usr/local/nvm/nvm.sh
 
 echo 'source /usr/local/nvm/nvm.sh' >> /etc/profile.d/nvm.sh
 ls -l /usr/local/nvm/nvm.sh
```
Install node
```
 # nvm ls-remote
 # NODE_VERSION=v6.11.1; echo ${NODE_VERSION} # Choose Latest LTS
 # nvm install ${NODE_VERSION}
 
 # unset http_proxy https_proxy
 # npm install elasticdump -g
```
아래는 아파치 서버를 기준으로 만든 elasticsearch template입니다.

Index Template
==============

**Index template example**
```
PUT _template/test-apache_template
{
  "template": "test_apache*",
  "settings": {
    "index":{
      "number_of_shards": 12,    
      "number_of_replicas": "1",
      "store": {
        "type": "niofs",
        "throttle": {
            "type": "none"
        }
      }
    }
  },
  "mappings": {
    "_default_": {
      "_source": {
        "enabled": true
      },
      "properties": {
        "request": {
          "type": "keyword"
        },
        "agent": {
          "type": "keyword"
        },
        "geoip": {
          "type": "geo_point"
        },
        "cookie": {
          "type": "keyword"
        },
        "auth": {
          "type": "keyword"
        },
        "ident": {
          "type": "keyword"
        },
        "verb": {
          "type": "keyword"
        },
        "message": {
          "type": "text"
        },
        "type": {
          "type": "keyword"
        },
        "tags": {
          "type": "keyword"
        },
        "path": {
          "type": "keyword"
        },
        "bytes": {
          "type": "long"
        },
        "clientip": {
          "type": "keyword"
        },
        "host": {
          "type": "keyword"
        },
        "responsetime": {
          "type": "long"
        },
        "httpversion": {
          "type": "keyword"
        },
        "timestamp": {
          "format":"dd/MMM/yyyy:HH:mm:ss Z",
          "type": "date"
        }
      }
    },
    "test_apache": {
      "_source": {
        "enabled": false
      },
      "properties": {
        "request": {
          "type": "keyword"
        },
        "agent": {
          "type": "keyword"
        },
        "geoip": {
          "type": "geo_point"
        },
        "cookie": {
          "type": "keyword"
        },
        "auth": {
          "type": "keyword"
        },
        "ident": {
          "type": "keyword"
        },
        "verb": {
          "type": "keyword"
        },
        "message": {
          "type": "text"
        },
        "type": {
          "type": "keyword"
        },
        "tags": {
          "type": "keyword"
        },
        "path": {
          "type": "keyword"
        },
        "bytes": {
          "type": "long"
        },
        "clientip": {
          "type": "keyword"
        },
        "host": {
          "type": "keyword"
        },
        "responsetime": {
          "type": "long"
        },
        "rawrequest": {
          "type": "text"
        },
        "timestamp": {
          "format":"dd/MMM/yyyy:HH:mm:ss Z",
          "type": "date"
        }
      }
    }
  }
} 
```
  

Use
===
```
elasticdump \
  --input=http://production.es.com:9200/my_index \
  --output=http://staging.es.com:9200/my_index \
  --type=data
```
템플릿을 Elasticsearch에 미리 등록 하는 것을 잊지마세요~

**Backgroud(as shell script)**
```
nohup elasticdump --input=http://sourceHost:9200/sourceIndex --output=http://targetHost:9200/targetIndex --type=data > /dev/null 2>&1 &
```
주의점
===

*   중계 서버에서 elasticdump를 실행해도 문제 없으나 네트워크!!!! 묵음
*   생각 보다 빠르지 않음, 약 20시간에(4 파라렐으로) 35,748,000 문서 정도 전송 되었음, 정송 크기로는 21.7기가 정도
*   디폴트 "--limit" 가 100인데. 늘리고 싶었으나, 누가 하지마~~ 해서 손안되었음, 아마 소스 es쪽의 쿼리양이 줄어들어서 소스 es 부하가 줄어 들것으로 생각됨 (지금 로드 졸라 올라간 상태)
*   ![](https://minsoo-jun.github.io/images/16252932.png)
*   평행은 상태는 4 파라렐으로 작업 마지막에 올라 온것은 하나더 추가해서 5 파라렐으로 변경.


