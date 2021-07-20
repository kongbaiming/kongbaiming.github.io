---
title: es启用xpack
---

## 配置TLS和身份验证

```shell
/usr/share/elasticsearch/bin/elasticsearch-certutil ca
/usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
#添加elasticsearch组权限
chgrp elasticsearch /etc/elasticsearch/elastic-certificates.p12 /etc/elasticsearch/elastic-stack-ca.p12
#修改文件权限640
chmod 640 /etc/elasticsearch/elastic-certificates.p12 /etc/elasticsearch/elastic-stack-ca.p12 
```

## elasticserch 配置文件

```
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/elastic-certificates.p12
```

## 重启elasticsearch 服务

```javascript
systemctl    restart   elasticsearch
```

## 配置身份验证

- elastic 提供两种方式创建身份验证

```javascript
#系统自动生成密码
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
#自定义密码
usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

- 将elasticsearch 密码添加至elasticsearch-keystore文件

```javascript
/usr/share/elasticsearch/bin/elasticsearch-keystore  add xpack.security.transport.ssl.keystore.secure_password
/usr/share/elasticsearch/bin/elasticsearch-keystore  add xpack.security.transport.ssl.truststore.secure_password
```

- 验证节点状态

```javascript
curl -u elastic:qZXo7EkxoxmKvDqQIwn5 http://192.168.99.185:9200/_cat/nodes?v
```

- elastic 节点之间交互需要通过证书,证书不一致会导致节点无法加入到集群，节点加入到集群后用户验证是通过elasticsearch-keystore文件进行身份验证

    拷贝证书文件和身份认证文件到其他elastic节点

```javascript
[root@elk-node1 elasticsearch]# scp elastic-certificates.p12  elastic-stack-ca.p12  elasticsearch.keystore  root@192.168.99.186:/etc/elasticsearch/
root@192.168.99.186's password:
```

- elastic 配置文件

```
cluster.name: elk-cluster
node.name: elk-node2
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 192.168.99.186
http.port: 9200
discovery.zen.ping.unicast.hosts: ["192.168.99.185", "192.168.99.186"]
discovery.zen.minimum_master_nodes: 1
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.monitoring.collection.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/elastic-certificates.p12
```

- 重启elasticsearch 服务

```javascript
systemctl    restart   elasticsearch
```

- 查看elasticsearch集群状态

```javascript
[root@elk-node1 /]# curl -u elastic:qZXo7EkxoxmKvDqQIwn5 http://192.168.99.185:9200/_cat/nodes?v
ip             heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.99.186           16          90  14    0.02    0.06     0.11 mdi       -      elk-node2
192.168.99.185           21          93  21    0.37    0.32     0.35 mdi       *      elk-node1
```

## **Kibana 配置**

```javascript
[root@elk-node1 /]# egrep  -v "*#|^$" /etc/kibana/kibana.yml 
server.port: 5601
server.host: "192.168.99.185"
server.name: "192.168.99.185"
elasticsearch.hosts: ["http://192.168.99.185:9200"]
kibana.index: ".kibana"
elasticsearch.username: "kibana"
elasticsearch.password: "cc29cgb2QcnheBQ9oOPX"
logging.quiet: true
i18n.locale: "zh-CN"
tilemap.url: 'http://webrd02.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}'
```

##  **Logstash 配置**

```javascript
[root@elk-node1 /]# cat /etc/logstash/conf.d/networklog.conf 
input {
  beats {
    port => 5044
  }
   
}

filter {
  if "huawei" in [tags] {
    grok{
      match => {"message" => "%{SYSLOGTIMESTAMP:time} %{DATA:hostname} %{GREEDYDATA:info}"}
        }
  }


   else if "h3c" in [tags] {
    grok{
      match => {"message" => "%{SYSLOGTIMESTAMP:time} %{YEAR:year} %{DATA:hostname} %{GREEDYDATA:info}"}
        }
  }
  else if "ruijie" in [tags] {
    grok{
      match => {"message" => "%{SYSLOGTIMESTAMP:time} %{DATA:hostname} %{GREEDYDATA:info}"}
        }
  }
mutate {
      add_field => [ "[zabbix_key]", "networklogs" ]
      add_field => [ "[zabbix_host]", "192.168.99.185" ]
      add_field => [ "count","%{hostname}%{info}" ]
      remove_field => ["message","time","year","offset","tags","path","host","@version","[log]","[prospector]","[beat]","[input][type]","[source]"]
    }
} 

output{
stdout{codec => rubydebug}
elasticsearch{
    index => "networklogs-%{+YYYY.MM.dd}"
    hosts => ["192.168.99.185:9200"]
    user => "elastic"
    password => "qZXo7EkxoxmKvDqQIwn5"    
    sniffing => false
    }
if [count]  =~ /(ERR|error|ERROR|Failed|failed)/ {
        zabbix {
                zabbix_host => "[zabbix_host]"
                zabbix_key => "[zabbix_key]"
                zabbix_server_host => "192.168.99.200"
                zabbix_server_port => "10051" 
                zabbix_value => "count"
    }
  }
}
```

## **配置head插件**

```javascript
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization,X-Requested-With,Content-Length,Content-Type
```

- 访问方式

```javascript
http://192.168.99.185/elasticsearch-head//?auth_user=elastic&auth_password=qZXo7EkxoxmKvDqQIwn5
```

