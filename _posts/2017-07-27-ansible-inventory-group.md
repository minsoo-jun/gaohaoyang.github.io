---
layout: post
title:  "ansible의 inventory파일에서 서버 group설정"
date:   2017-07-27
categories: ansible
tags: ansible elasticsearch kibana
excerpt: 하나의 inventory에서 복수의 servers를 설정하고 group으로 실행하면서 각 servers만을 특정해서 실행하기
mathjax: true
---
Ansible > inventory
===================
하나의 inventory에서 복수의 servers를 설정하고 group으로 실행하면서 각 servers만을 특정해서 실행하기.
예제의 시나리오는 elasticsearch와 kibana에 x-pack을 설치하는 플레이북 입니다.
### 전체 디렉터리 구조
```bash
$> pwd;find . | sort | sed '1d;s/^\.//;s/\/\([^/]*\)$/|--\1/;s/\/[^/|]*/|  /g'

|--hosts
|  |--es-servers
|  |--kibana-servers
|  |--logplatform-servers
|--install
|  |--group_vars
|  |  |--all
|  |  |--es-servers
|  |  |--kibana-servers
|  |--install_es.yml
|  |--install_kibana.yml
|  |--install_xpack.yml
|  |--README.md
|  |--roles
|  |  |--es
|  |  |--es_startstop
|  |  |  |--tasks
|  |  |  |  |--es_shutdown.yml
|  |  |  |  |--es_start.yml
|  |  |  |  |--main.yml
|  |  |  |--tasks
|  |  |  |  |--download_unpack.yml
|  |  |  |  |--es_setting.yml
|  |  |  |  |--main.yml
|  |  |  |  |--os_setting.yml
|  |  |  |  |--plugin_head.yml
|  |  |  |--templates
|  |  |  |  |--elasticsearch.yml.j2
|  |  |  |  |--shutdown.sh.j2
|  |  |  |  |--startup.sh.j2
|  |  |--kibana
|  |  |--kibana_startstop
|  |  |  |--tasks
|  |  |  |  |--kibana_shutdown.yml
|  |  |  |  |--kibana_start.yml
|  |  |  |  |--main.yml
|  |  |  |--tasks
|  |  |  |  |--download_unpack.yml
|  |  |  |  |--kibana_setting.yml
|  |  |  |  |--main.yml
|  |  |  |--templates
|  |  |  |  |--kibana.yml.j2
|  |  |  |  |--shutdown.sh.j2
|  |  |  |  |--startup.sh.j2
|  |  |--xpack
|  |  |  |--tasks
|  |  |  |  |--main.yml
```

### Inventory file : logplatform

이번에 대상으로 하는 인벤토리 파일은 "logplatform-servers"입니다.
파일의 내용은 아래와 같습니다.
```yaml
[logplatform-servers:children]
es-servers
kibana-servers
 
[es-servers]
192.168.56.101 IPV4_ADDR="192.168.56.101" NODE_NAME="seoungbin" HEAP_SIZE=1g
 
[kibana-servers]
192.168.56.104 IPV4_ADDR="192.168.56.104"
```
### Debug playbook
확인용 플레이북으로 정상적으로 서버를 가지고 오는지 확인해 합니다. 
```yaml
- name: Include vars of es-servers
  include_vars: "../../../group_vars/es-servers"
 
- debug:
    msg: "ES Home: {{ ES_HOME_DIR }}, Inventory Hostname: {{ inventory_hostname }}, GroupName: {{ group_names }}"
 
- debug:
    msg: "GroupName: {{ item }}"
  with_items:
    - "{{ group_names }}"
  when: item == 'es-servers'
  register: target_server
 
- debug:
    msg: "Stats = {{ target_server.results[0].item }}"
```
실행 결과 확인
```bash
[jun@ansible server]$ ansible-playbook -i hosts/logplatform-servers -k -K install/install_xpack.yml -e "server=logplatform opt_proxy=1" -v
Using /etc/ansible/ansible.cfg as config file
SSH password:
SUDO password[defaults to SSH password]:PLAY [all] *********************************************************************TASK [setup] *******************************************************************
ok: [192.168.56.104]
ok: [192.168.56.101]TASK [xpack : Include vars of es-servers] **************************************
ok: [192.168.56.101] => {"ansible_facts": {"ES_CLUSTER_NAME": "minsoo", "ES_DATA_PATH": "/elastic/elasticsearch", "ES_DOWNLOAD_URL": "https://artifacts.elastic.co/downloads/elasticsearch/{{ ES_DOWNLOAD_VERSION }}.tar.gz", "ES_DOWNLOAD_VERSION": "elasticsearch-5.5.0", "ES_GROUP": "elasticsearch", "ES_HOME_BASIS": "/usr/local/", "ES_HOME_DIR": "/usr/local/elasticsearch", "ES_MEMORY_LOCK": "false", "ES_OWNER": "elasticsearch", "ES_SERVER_NODES_INFO": [{"node": "01", "port": 9200, "tcp_port": 9300}, {"node": "02", "port": 9201, "tcp_port": 9301}], "NEXUS_3RD_URL": "http://stg-qtrrepo101z.stg.jp.local/service/local/repositories/thirdparty/content/"}, "changed": false}
ok: [192.168.56.104] => {"ansible_facts": {"ES_CLUSTER_NAME": "minsoo", "ES_DATA_PATH": "/elastic/elasticsearch", "ES_DOWNLOAD_URL": "https://artifacts.elastic.co/downloads/elasticsearch/{{ ES_DOWNLOAD_VERSION }}.tar.gz", "ES_DOWNLOAD_VERSION": "elasticsearch-5.5.0", "ES_GROUP": "elasticsearch", "ES_HOME_BASIS": "/usr/local/", "ES_HOME_DIR": "/usr/local/elasticsearch", "ES_MEMORY_LOCK": "false", "ES_OWNER": "elasticsearch", "ES_SERVER_NODES_INFO": [{"node": "01", "port": 9200, "tcp_port": 9300}, {"node": "02", "port": 9201, "tcp_port": 9301}], "NEXUS_3RD_URL": "http://stg-qtrrepo101z.stg.jp.local/service/local/repositories/thirdparty/content/"}, "changed": false}TASK [xpack : debug] ***********************************************************
ok: [192.168.56.101] => {
    "msg": "ES Home: /usr/local/elasticsearch, Inventory Hostname: 192.168.56.101, GroupName: [u'es-servers', u'logplatform-servers']"
}
ok: [192.168.56.104] => {
    "msg": "ES Home: /usr/local/elasticsearch, Inventory Hostname: 192.168.56.104, GroupName: [u'kibana-servers', u'logplatform-servers']"
}TASK [xpack : debug] ***********************************************************
skipping: [192.168.56.101] => (item=logplatform-servers)  => {"changed": false, "item": "logplatform-servers", "skip_reason": "Conditional check failed", "skipped": true}
ok: [192.168.56.101] => (item=es-servers) => {
    "item": "es-servers",
    "msg": "GroupName: es-servers"
}
skipping: [192.168.56.104] => (item=logplatform-servers)  => {"changed": false, "item": "logplatform-servers", "skip_reason": "Conditional check failed", "skipped": true}
skipping: [192.168.56.104] => (item=kibana-servers)  => {"changed": false, "item": "kibana-servers", "skip_reason": "Conditional check failed", "skipped": true}TASK [xpack : debug] ***********************************************************
ok: [192.168.56.101] => {
    "msg": "Stats = es-servers"
}
ok: [192.168.56.104] => {
    "msg": "Stats = kibana-servers"
}PLAY RECAP *********************************************************************
192.168.56.101             : ok=5    changed=0    unreachable=0    failed=0
192.168.56.104             : ok=4    changed=0    unreachable=0    failed=0
```

### 전체 playbook
```yaml
#import vars from es-servers and kibana-servers
- name: Include vars of es-servers
  include_vars: "../../../group_vars/es-servers"
 
- name: Include vars of es-servers
  include_vars: "../../../group_vars/kibana-servers"
 
- debug:
    msg: "GroupName: {{ item }}"
  with_items:
    - "{{ group_names }}"
  when: item == 'es-servers'
  register: target_server
 
- debug:
    msg: "Stats = {{ target_server.results[0].item }}"
 
# stop es
- include: ../../es_startstop/tasks/es_shutdown.yml
  ignore_errors: True
  when: "'es-servers' == target_server.results[0].item"
 
- name: remove xpack
  command: "{{ ES_HOME_DIR }}_{{ item.node }}/bin/elasticsearch-plugin remove x-pack"
  become_user: "{{ ES_OWNER }}"
  environment: "{{ PROXY_ENV }}"
  ignore_errors: True
  with_items:
    - "{{ ES_SERVER_NODES_INFO }}"
  when: "'es-servers' == target_server.results[0].item"
 
- name: install xpack
  command: "{{ ES_HOME_DIR }}_{{ item.node }}/bin/elasticsearch-plugin install x-pack"
  become_user: "{{ ES_OWNER }}"
  environment: "{{ PROXY_ENV }}"
  ignore_errors: True
  with_items:
    - "{{ ES_SERVER_NODES_INFO }}"
  when: "'es-servers' == target_server.results[0].item"
 
# start elasticsearch
- include: ../../es_startstop/tasks/es_start.yml
  when: "'es-servers' == target_server.results[0].item"
 
- debug:
    msg: "GroupName: {{ item }}"
  with_items:
    - "{{ group_names }}"
  when: item == 'kibana-servers'
  register: target_kibana_server
 
- debug:
    msg: "Stats = {{ target_kibana_server.results[0].item }}"
 
# stop kibana
- include: ../../kibana_startstop/tasks/kibana_shutdown.yml
  ignore_errors: True
  when: "'kibana-servers' == target_kibana_server.results[0].item"
 
- name: install xpack
  command: "{{ KIBANA_HOME_DIR }}/bin/kibana-plugin remove x-pack"
  become_user: "{{ ES_OWNER }}"
  environment: "{{ PROXY_ENV }}"
  ignore_errors: True
  when: "'kibana-servers' == target_kibana_server.results[0].item"
 
- name: install xpack
  command: "{{ KIBANA_HOME_DIR }}/bin/kibana-plugin install x-pack"
  become_user: "{{ ES_OWNER }}"
  environment: "{{ PROXY_ENV }}"
  ignore_errors: True
  when: "'kibana-servers' == target_kibana_server.results[0].item"
 
# start kibana
- include: ../../kibana_startstop/tasks/kibana_start.yml
  when: "'kibana-servers' == target_kibana_server.results[0].item"
 
#end
```
<span style="color:red">`주의!! when: "'kibana-servers' == target_kibana_server.results[0].item" 에서 when 안에서는 변수를 표현 할 때는 {{ }} 를 쓰지 않는다`</span>


`when 에서 비교 할 때는 전체를 " " 로 묶고 안에서 처리해야 된다`