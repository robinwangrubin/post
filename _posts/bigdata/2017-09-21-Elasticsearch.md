---
layout: post
title: Elasticsearch
categories: BigData
---

## 架构原理

### segment buffer translog


### segment merge


### routing replica


### shard

--------------------------------

## 安装 elasticsearch

```
useradd elasticsearch -M -s /sbin/nologin
https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.4.6/elasticsearch-2.4.6.tar.gz
tar xf elasticsearch-2.4.6.tar.gz -C /usr/local/
chown -R elasticsearch: /usr/local/elasticsearch-2.4.6
ln -s /usr/local/elasticsearch-2.4.6 /usr/local/elasticsearch
```

## 安装 plugin

```
cd /usr/local/elasticsearch
./bin/plugin install mobz/elasticsearch-head
```

## 配置 elasticsearch cluster

```
mv /usr/local/elasticsearch/config/elasticsearch.yml /usr/local/elasticsearch/config/elasticsearch.yml.bak
cat >/usr/local/elasticsearch/config/elasticsearch.yml <<EOF
 cluster.name: robin_test_cluster
 node.name: xxxxYour Hostname	
 network.host: 0.0.0.0
 index.number_of_shards 5
 index.number_of_replicas 2
 discovery.zen.minimum_master_nodes: 3
 discovery.zen.ping_timeout: 100s
 discovery.zen.fd.ping_timeout: 100s
 discovery.zen.ping.multicast.enabled: false
 discovery.zen.ping.unicast.hosts: ["192.168.1.190","192.168.1.191","192.168.1.192"]
EOF
```

## Supervisor 守护

```
cat >/etc/supervisor/conf/elasticsearch.ini <<EOF
[program:Elasticsearch]
command = /usr/local/elasticsearch/bin/elasticsearch
autostart = true
autorestart = true
startsecs = 3
startretries = 3
stopwaitsecs = 5
user = elasticsearch
redirect_stderr = true
stdout_logfile = /var/log/supervisor/elasticsearch.log
stdout_logfile_maxbytes = 500MB
stdout_logfile_backups = 0
EOF
```

### 启动es服务

```
supervisorctl update
```

### 服务验证

```
netstat -lntup|grep 9200
netstat -lntup|grep 9300
```

```
curl 'http://localhost:9200/?pretty'
{
  "name" : "Abner Little",
  "cluster_name" : "RobinES",
  "cluster_uuid" : "qFXwzSfgRvSVv8EhJmeSlQ",
  "version" : {
    "number" : "2.4.6",
    "build_hash" : "5376dca9f70f3abef96a77f4bb22720ace8240fd",
    "build_timestamp" : "2017-07-18T12:17:44Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.4"
  },
  "tagline" : "You Know, for Search"
}
```

-------------------------------------

## Elasticsearch日常操作


### 增


```
curl -XPOST http://localhost:9200/logstash-2015.06.21/testlog -d '{"date" : "1434966686000","user" : "chenlin7","mesg" : "first message into Elasticsearch"}'
{"_index":"logstash-2015.06.21","_type":"testlog","_id":"AV6eVtpCdKKfa5USSXuh","_version":1,"_shards":{"total":2,"successful":2,"failed":0},"created":true}
```

### 查

```
curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/AV6eVtpCdKKfa5USSXuh?pretty
{
  "_index" : "logstash-2015.06.21",
  "_type" : "testlog",
  "_id" : "AV6eVtpCdKKfa5USSXuh",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "date" : "1434966686000",
    "user" : "chenlin7",
    "mesg" : "first message into Elasticsearch"
  }
}

curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/AV6eVtpCdKKfa5USSXuh/_source?pretty
{
  "date" : "1434966686000",
  "user" : "chenlin7",
  "mesg" : "first message into Elasticsearch"
}

curl -XGET 'http://127.0.0.1:9200/logstash-2015.06.21/testlog/AV6eVtpCdKKfa5USSXuh?fields=user,mesg&pretty'
```



### 改

```
curl -XPOST http://localhost:9200/logstash-2015.06.21/testlog/AV6eVtpCdKKfa5USSXuh/_update -d '{"doc": {"date" : "1111111111111","user" : "someone","mesg" : "first message into Elasticsearch"}}'
curl -XPOST http://localhost:9200/logstash-2015.06.21/testlog/AV6eVtpCdKKfa5USSXuh -d '{"date" : "2222222222","user" : "Robin","mesg" : "first message into Elasticsearch version 2"}'
```

### 删

```
curl -XDELETE http://localhost:9200/logstash-2015.06.21/testlog/AV6eVtpCdKKfa5USSXuh
curl -XDELETE http://localhost:9200/logstash-2015.06*
```

-----------------------------

`上面都是对单条数据的操作；下面开始做全文搜索和聚合请求`

## 全文搜索

```
curl -XPOST http://localhost:9200/logstash-2015.06.21/testlog -d '{"date" : "1110","user" : "Robin","mesg" : "first message into Elasticsearch"}'
curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=first
curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=mesg:first
curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=user:"Robin"
curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=_exists_:user
curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=_missing_:user
curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=user:"chenlin?"
curl -XGET http://127.0.0.1:9200/logstash-2015.06.21/testlog/_search?q=frist~
```


## 聚合请求

* 下载测试数据

```
wget http://wangrubin.com/testfile/accounts.json
```

* 测试数据样例

```
{"account_number":372,"balance":28566,"firstname":"Alba","lastname":"Forbes","age":24,"gender":"M","address":"814 Meserole Avenue","employer":"Isostream","email":"albaforbes@isostream.com","city":"Clarence","state":"OR"}
```


* 写入测试数据到es中

```
curl -XPOST 'localhost:9200/bank/account/_bulk?pretty' --data-binary  @accounts.json
```

* 验证是否写入成功

```
curl 'localhost:9200/_cat/indices?v'
health status index               pri rep docs.count docs.deleted store.size pri.store.size 
green  open   logstash-2017.09.20   5   1          1            0      8.4kb          4.2kb 
green  open   bank                  5   1       1000            0    925.2kb        456.9kb 
```

* 搜索全部的文档

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match_all": {} }
}'
```

* 搜索back索引下第一条数据

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match_all": {} },
  "size": 1
}'
```

* 下面的命令请求了第10-20的文档

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match_all": {} },
  "from": 10,
  "size": 10
}'
```

* 指定了返回特定的字段

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match_all": {} },
  "_source": ["account_number", "balance"]
}'
```

* 返回特定的字段

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match": { "account_number": 20 } }
}'
```

* 查询地址为mill的信息

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match": { "address": "mill" } }
}'
```

* 查询地址为mill或者lane的信息

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match": { "address": "mill lane" } }
}'
```

* 如果我们想要返回同时包含mill和lane的，可以通过match_phrase查询

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": { "match_phrase": { "address": "mill lane" } }
}'
```

* 比如查询同时包含mill和lane的文档

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}'
```

* 查询包含mill或者lane的文档

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": {
    "bool": {
      "should": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}'
```

* 排除包含mill和lane的文档

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must_not": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}'
```

* bool查询可以同时使用must, should, must_not组成一个复杂的查询

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}'
```

* 查询在2000-3000范围内的所有文档

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}'
```

* 数据聚合统计

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state"
      }
    }
  }
}'
```

* 统计不同账户状态下的平均余额

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}'
```

* 先按范围分组，在统计不同性别的账户余额

```
curl -XPOST 'localhost:9200/bank/_search?pretty' -d '
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}'
```

* 模拟统计生产环境的每小时日志量

```
#!/usr/bin/env bash
int=1
while(( $int<=1000 ))
do
y=`date +%Y`
m=`date +%m`
d=`date +%d`
h=`date +%H`
M=`date +%M`
s=`date +%S`
cat >>/root/elasticsearch.json<<EOF
{"index":{"_id":"$int"}}
{"account_number":1,"balance":39225,"firstname":"Amber","lastname":"Duke","age":32,"gender":"M","address":"880 Holmes Lane","employer":"Pyrami","email":"amberduke@pyrami.com","city":"Brogan","state":"IL","localtime":"${d}/${m}/${y}:${h}:${M}:${s} +0800"}
EOF
sleep 3
let int=int+1
done
```

```
curl -XPOST 'localhost:9200/wangrubin-2017-09-21/account/_bulk?pretty' --data-binary  @elasticsearch.json
```


```
curl -XPOST 'localhost:9200/wangrubin*/_search?pretty' -d '
{
  "query": { "match_phrase": { "localtime": "21 09 2017 12" } },
  "size": 0
}'
```

---------------------------------------


## Reindex

```
curl -XPOST http://localhost:9200/_reindex -d '
{
  "source": {
    "index": "logstash-2015.06.21"
  },
  "dest": {
    "index": "logstash-2017.09.20"
  }
}'
```
经过测试这功能就像 cp src dst

--------------------------------

## 性能优化

```
pip install elasticsearch-curator
```

```
curator --timeout 36000 --host 10.0.0.100 delete indices --older-than 5 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-mweibo-nginx-
curator --timeout 36000 --host 10.0.0.100 delete indices --older-than 10 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-mweibo-client- --exclude 'logstash-mweibo-client-2015.05.11'
curator --timeout 36000 --host 10.0.0.100 delete indices --older-than 30 --time-unit days --timestring '%Y.%m.%d' --regex '^logstash-mweibo-\d+'
curator --timeout 36000 --host 10.0.0.100 close indices --older-than 7 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-
curator --timeout 36000 --host 10.0.0.100 optimize --max_num_segments 1 indices --older-than 1 --newer-than 7 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-
```
logstash-mweibo-nginx-yyyy.mm.dd 索引保存最近 5 天

logstash-mweibo-client-yyyy.mm.dd 保存最近 10 天

logstash-mweibo-yyyy.mm.dd 索引保存最近 30 天

所有七天前的 logstash-* 索引都暂时关闭不用

最后对所有非当日日志做 segment 合并优化。

----------------------------------

## 访问控制

* 安装插件

```
cd /usr/local/elasticsearch
./bin/plugin install elasticsearch/license/latest
./bin/plugin install elasticsearch/shield/latest
```

* 重启es

``` 
supervisorctl restart Elasticsearch
```

* 创建用户

```
./bin/shield/esusers useradd elasticsearch -r admin
```

* 查看用户列表

```
./bin/shield/esusers list
```

* 修改用户密码

```
./bin/shield/esusers passwd root
```

* 删除用户

```
./bin/shield/esusers userdel root
```

* 查看所有可用命令

```
./bin/shield/esusers -h
```




------------------------------------

## es集群监控

* 集群层面监控

```
curl -u elasticsearch:xiaoaojianghu -XGET '127.0.0.1:9200/_cluster/health?pretty'
{
  "cluster_name" : "robin_test_cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 1,
  "active_shards" : 2,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

* 索引层面监控

```
curl -u elasticsearch:xiaoaojianghu -XGET '127.0.0.1:9200/_cluster/health?level=indices&pretty'
{
  "cluster_name" : "robin_test_cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 1,
  "active_shards" : 2,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0,
  "indices" : {
    ".kibana" : {
      "status" : "green",
      "number_of_shards" : 1,
      "number_of_replicas" : 1,
      "active_primary_shards" : 1,
      "active_shards" : 2,
      "relocating_shards" : 0,
      "initializing_shards" : 0,
      "unassigned_shards" : 0
    }
  }
}
```


* 节点层面监控

```
curl -u elasticsearch:xiaoaojianghu -XGET 127.0.0.1:9200/_nodes/stats?pretty
curl -u elasticsearch:xiaoaojianghu -XGET 'http://127.0.0.1:9200/_nodes/_local/hot_threads?interval=60s'
curl -u elasticsearch:xiaoaojianghu -XGET 'http://127.0.0.1:9200/_nodes/_local/hot_threads?type=wait&interval=60s'
curl -u elasticsearch:xiaoaojianghu -XGET 'http://127.0.0.1:9200/_nodes/_local/hot_threads?type=block&interval=60s'
````



* 查看集群等待任务数

```
curl -u elasticsearch:xiaoaojianghu  -XGET 'http://127.0.0.1:9200/_cluster/pending_tasks?pretty'
```

`shell中获取状态`

```
* /_cat/nodes
* /_cat/shards
* /_cat/shards/{index}
* /_cat/aliases
* /_cat/aliases/{alias}
* /_cat/tasks
* /_cat/master
* /_cat/plugins
* /_cat/fielddata
* /_cat/fielddata/{fields}
* /_cat/pending_tasks
* /_cat/count
* /_cat/count/{index}
* /_cat/snapshots/{repository}
* /_cat/recovery
* /_cat/recovery/{index}
* /_cat/segments
* /_cat/segments/{index}
* /_cat/thread_pool
* /_cat/thread_pool/{thread_pools}/_cat/nodeattrs
* /_cat/allocation
* /_cat/repositories
* /_cat/health
* /_cat/indices
* /_cat/indices/{index}
```



```
curl -u elasticsearch:xiaoaojianghu  -XGET 'http://127.0.0.1:9200//_cat/nodes'
192.168.1.191 192.168.1.191 5 37 0.01 d * hadoop-slave01 
192.168.1.192 192.168.1.192 6 32 0.00 d m hadoop-slave02 
192.168.1.190 192.168.1.190 5 43 0.00 d m hadoop-master
```

```
curl -u elasticsearch:xiaoaojianghu  -XGET 'http://127.0.0.1:9200/_cat/health'
1506059940 13:59:00 robin_test_cluster green 3 3 12 6 0 0 0 0 - 100.0% 
```

```
curl -u elasticsearch:xiaoaojianghu  -XGET 'http://127.0.0.1:9200/_cat/nodes?v'
host          ip            heap.percent ram.percent load node.role master name           
192.168.1.191 192.168.1.191            5          37 0.00 d         *      hadoop-slave01 
192.168.1.192 192.168.1.192            6          32 0.00 d         m      hadoop-slave02 
192.168.1.190 192.168.1.190            5          43 0.00 d         m      hadoop-master
```

```
curl -u elasticsearch:xiaoaojianghu  -XGET 'http://127.0.0.1:9200/_cat/nodes?v&h=ip,port,heapPercent,heapMax'
ip            port heapPercent  heapMax 
192.168.1.191 9300           5 1015.6mb 
192.168.1.192 9300           6 1015.6mb 
192.168.1.190 9300           5 1015.6mb 
```

```
curl -u elasticsearch:xiaoaojianghu  -XGET 'http://127.0.0.1:9200/_cat/nodes?help'

id                               | id,nodeId                          | unique node id                                                                                                   
pid                              | p                                  | process id                                                                                                       
host                             | h                                  | host name                                                                                                        
ip                               | i                                  | ip address                                                                                                       
port                             | po                                 | bound transport port                                                                                             
version                          | v                                  | es version                                                                                                       
build                            | b                                  | es build hash         
```

```
curl -XGET http://127.0.0.1:9200/_cat/nodes?v&h=i,po,hp,hm
```


* 查看索引分片状态

```
curl -u elasticsearch:xiaoaojianghu  -XGET 'http://127.0.0.1:9200/_cat/shards?v'  
```



* 查看当前数据恢复状态

```
curl -u elasticsearch:xiaoaojianghu  -XGET 'http://127.0.0.1:9200/_cat/recovery?active_only&v&h=i,s,shost,thost,fp,bp,tr,trp,trt'
```

* 线程池状态

```
curl -u elasticsearch:xiaoaojianghu  -XGET 'http://127.0.0.1:9200/_cat/thread_pool?v'     
```
--------------------

end
