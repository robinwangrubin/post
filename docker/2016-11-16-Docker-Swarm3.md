---
layout: post
title: "Docker 1.12 Swarm集群实战 第三章"
categories: Docker
---


* content
{:toc}

## 前言

上一章我们搭建并测试docker swarm集群；这一章将我们的docker币应用部署到swarm集群上

* 在swarm集群上运行docker币应用

在第一章的我们在单节点环境下运行docker币应用的时候, 我们build的应用镜像image是保存在本地的.

现在我们使用swarm集群来运行,docker币服务分布在很多不同的服务器节点上, 难道每个节点都要build镜像, 或者要把image拷贝到别的节点上?

当然不用, 我们用本地镜像仓库docker registry来保存image. 每个节点从registry拉取需要的image就可以了.

----------------------

## 部署docker私有仓库registry

{% highlight shell %}
[root@node0 ~]# docker service create --name registry -p 5000:5000 --constraint 'node.hostname==node0' registry
{% endhighlight %}

\-\-constraint：限制registry只运行在node0上；防止重启服务后registry服务被分配到别的节点上；

生产环境务必通过-v参数挂载本地卷用以保存镜像；

> Note：我会在后面的章节单独补充这个问题

{% highlight shell %}
[root@node0 ~]# docker service ls
ID            NAME      REPLICAS  IMAGE     COMMAND
cgk5fahusjsy  registry  1/1       registry  
[root@node0 ~]# docker service ps registry
ID                         NAME        IMAGE     NODE   DESIRED STATE  CURRENT STATE           ERROR
0s4yz56rizgp7pi2e8vlk9t97  registry.1  registry  node0  Running        Running 11 seconds ago  
{% endhighlight %}


在任意一台服务器上验证registry是否部署成功

{% highlight shell %}
[root@node0 ~]# curl localhost:5000/v2/_catalog
{"repositories":[]}
{% endhighlight %}

服务工作正常


### 测试registry私有仓库

查看下我们现有的镜像, 选取一个例如alpine, 如果想把alpine镜像推送到本地registry

{% highlight shell %}
[root@node0 ~]# docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
dockercoins_webui    latest              121d059a6b00        19 hours ago        209.7 MB
dockercoins_rng      latest              5ca4119bc4ba        19 hours ago        98.98 MB
dockercoins_hasher   latest              57e93a785b7d        19 hours ago        301 MB
dockercoins_worker   latest              35725037c570        20 hours ago        96.01 MB
node                 4-slim              b2bd4c48b5dc        3 days ago          205.3 MB
redis                latest              5f515359c7f8        4 days ago          182.9 MB
nginx                latest              05a60462f8ba        4 days ago          181.4 MB
ruby                 alpine              35562b355c05        11 days ago         126.7 MB
python               alpine              8dd7712cca84        12 days ago         88.49 MB
registry             latest              c9bd19d022f6        3 weeks ago         33.27 MB
alpine               latest              baa5d63471ea        3 weeks ago         4.799 MB
{% endhighlight %}


需要先tag这个镜像的名字成/:


{% highlight shell %}
[root@node0 ~]# docker tag alpine localhost:5000/busybox
[root@node0 ~]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
...
alpine                   latest              baa5d63471ea        3 weeks ago         4.799 MB
localhost:5000/busybox   latest              baa5d63471ea        3 weeks ago         4.799 MB
{% endhighlight %}


下面就可以使用docker push 推送image了

{% highlight shell %}
[root@node0 ~]# docker push localhost:5000/busybox
The push refers to a repository [localhost:5000/busybox]
011b303988d2: Pushed 
latest: digest: sha256:1354db23ff5478120c980eca1611a51c9f2b88b61f24283ee8200bf9a54f2e5c size: 528
{% endhighlight %}

push成功后, 可以调用registry API查看 registry中的镜像

{% highlight shell %}
[root@node0 ~]# curl localhost:5000/v2/_catalog
{"repositories":["busybox"]}
{% endhighlight %}

### 推送docker币镜像到registry

现在我们要做的是, 把build完成的,docker币的镜像推送到本地镜像库中, 这样我们用swram集群在多个节点部署应用的时候就不需要在build,可以直接从本地镜像库中拉取镜像了.

* 具体步骤:

1. build docker币镜像(我们已经在node01上build过了, 你也可以直接使用这些镜像.)
2. retag镜像名字成 localhost:5000/:名字
3. push镜像到我们的本地镜像库

{% highlight shell %}
[root@node0 ~]# docker tag dockercoins_hasher localhost:5000/dockercoins_hasher:v0.1
[root@node0 ~]# docker push localhost:5000/dockercoins_hasher
The push refers to a repository [localhost:5000/dockercoins_hasher]
b47ebe6f58ef: Pushed 
d74e7d988f6c: Pushed 
14883ca5915d: Pushed 
5b0f47d537aa: Pushed 
a1e628dbe64d: Pushed 
d6b54565786a: Pushed 
399d4c9ce74e: Pushed 
5c0a4f4b3a35: Pushed 
011b303988d2: Mounted from busybox 
v0.1: digest: sha256:17458c3a7c16592414838b1ef9c044e8e8d5e22171e16579ac634f3363689bbb size: 2205
[root@node0 ~]# curl localhost:5000/v2/_catalog
{"repositories":["busybox","dockercoins_hasher"]}
{% endhighlight %}

我们的docker币应用一共需要5个service：hasher, rng, webui, worker, redis

hasher推送完了, 你可以使用相同的步骤完成rng, webui, worker的推送.

简单起见, 你可以使用下面的一段命令, 完成整个过程, 直接贴到命令窗口执行就可以了.

{% highlight shell %}
DOCKER_REGISTRY=localhost:5000
TAG=v0.1
for SERVICE in rng webui worker; do
  docker tag dockercoins_$SERVICE $DOCKER_REGISTRY/dockercoins_$SERVICE:$TAG
  docker push $DOCKER_REGISTRY/dockercoins_$SERVICE
done
{% endhighlight %}

* 验证仓库中有哪些镜像

{% highlight shell %}
[root@node0 ~]# curl localhost:5000/v2/_catalog
{"repositories":["busybox","dockercoins_hasher","dockercoins_rng","dockercoins_webui","dockercoins_worker"]}
{% endhighlight %}

-------------------

## overlay网络

为了让我们的应用跑在swram集群上,我们还需要解决容器间的网络访问问题.

单台服务器的时候我们所有的应用容器都跑在一台主机上, 所以容器之之间的网络是互通的.

现在我们的集群有5台主机, 所以docker币应用的服务会分布在这5台主机上.

如何保证不同主机上的容器网络互通呢?

swarm集群已经帮我们解决了这个问题了,就是用overlay network.

在docker 1.12以前, swarm集群需要一个额外的key-value存储(consul, etcd etc). 来同步网络配置, 保证所有容器在同一个网段中.

但在docker 1.12已经内置了这个存储, 集成了overlay networks的支持.

下面我们演示下如何创建一个overlay network

为我们的docker币应用创建一个名为dockercoins的overlay network


{% highlight shell %}
[root@node0 ~]# docker network create --driver overlay dockercoins
8qvpdmja03td9dui3njmad822
[root@node0 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
733ea09f32b0        bridge              bridge              local               
6e4c3a948f31        docker_gwbridge     bridge              local               
8qvpdmja03td        dockercoins        overlay             swarm               
1ea04b17201b        host                host                local               
b87rqkh1nkl1        ingress             overlay             swarm               
f11c57b22740        none                null                local      
{% endhighlight %}

在网络列表中你可以看到dockercoins网络的SCOPE是swarm, 表示该网络在整个swarm集群生效的, 其他一些网络是local, 表示本机网络.

你只需要在manager节点创建network, swarm集群会自动处理配置到其他的节点,这时你可以查看其他节点的network. dockercoins网络已经都创建了

-------------------

## 在网络上运行容器

现在我们有了dockercoins网络了, 怎么指定容器或者服务运行在哪个网络上呢？

下面我们可以先启动一个redis服务, 作为我们docker币应用的数据库.

{% highlight shell %}
[root@node0 ~]# docker service create --network dockercoins --name redis redis
3l07xdfj4rjkcx9ak713eiwdo
[root@node0 ~]# docker service ls      
ID            NAME      REPLICAS  IMAGE     COMMAND
3l07xdfj4rjk  redis     1/1       redis     
cgk5fahusjsy  registry  1/1       registry  
[root@node0 ~]# docker service ps redis
ID                         NAME     IMAGE  NODE   DESIRED STATE  CURRENT STATE               ERROR
3nlxouh6ma6234w7kfyzqtwru  redis.1  redis  node4  Running        Running about a minute ago  
{% endhighlight %}

## 在swarm集群上运行docker币应用

redis服务创建好了, 下面我们使用之前push到本地镜像仓库的镜像启动hasher, rng, webui, worker服务, 以hasher为例:


{% highlight shell %}
[root@node0 ~]# docker service create --network dockercoins --name hasher localhost:5000/dockercoins_hasher:v0.1
22cli9o3u9hoipweqoxeric7a
[root@node0 ~]# docker service ls
ID            NAME      REPLICAS  IMAGE                                   COMMAND
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1  
3l07xdfj4rjk  redis     1/1       redis                                   
cgk5fahusjsy  registry  1/1       registry
{% endhighlight %}

下面的这段脚本可以帮我们启动所有的服务, 你可以直接粘贴到命令行执行:

{% highlight shell %}
DOCKER_REGISTRY=localhost:5000
TAG=v0.1
for SERVICE in hasher rng webui worker; do
docker service create --network dockercoins --name $SERVICE \
       $DOCKER_REGISTRY/dockercoins_$SERVICE:$TAG
done
{% endhighlight %}

完成后检查我们的servcie启动情况

{% highlight shell %}
[root@node0 ~]# docker service ls
ID            NAME      REPLICAS  IMAGE                                   COMMAND
0pcuhiw9jmu4  rng       1/1       localhost:5000/dockercoins_rng:v0.1     
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1  
3l07xdfj4rjk  redis     1/1       redis                                   
7qxzlamgstd9  webui     1/1       localhost:5000/dockercoins_webui:v0.1   
cgk5fahusjsy  registry  1/1       registry                                
clvgacmx24p7  worker    1/1       localhost:5000/dockercoins_worker:v0.1  
{% endhighlight %}

ok，现在我们可以继续挖docker币了，打开webui

对了，我们忘记发布webui的端口了，所以没法访问webui，因为无法动态修改service端口发布，所以只能删除webui服务，重新构建一个

{% highlight shell %}
[root@node0 ~]# docker service rm webui
webui
[root@node0 ~]# docker service create --network dockerconins --name webui --publish 8000:80 localhost:5000/dockercoins_webui:v0.1
52c93icugean8toaglopnxuok
{% endhighlight %}

好了，现在可用浏览器访问所有节点的8000端口

![1]({{ site.url }}/pic/docker-swarm-3/3fde9911-a1ba-4874-86c5-7c261302c18c.png)


----------------------

## 扩展(Scaling)应用

上面我们分析了docker币应用的瓶颈, 在单台服务器的情况下, 我们scale up worker节点, 到10个副本的时候. 产生的docker币数量并没有按照预想的情况增加.

我们找到了两个瓶颈:

1. 单台服务器性能的瓶颈.
2. rng 服务的瓶颈.

其实主要是rng服务的瓶颈，为了解决这个性能问题, 我们引入了docker swarm集群.

* 增加worker容器到10个

{% highlight shell %}
[root@node0 ~]# docker service scale worker=10
{% endhighlight %}

前面我们已经分析过瓶颈出现在rng服务, 下面我们就来扩展rng服务.

当然我们可以使用docker service scale命令;跟上面一样增加rng服务的容器.

不过这次我们换一种方式,这次我们通过修改rng服务的属性实现.

使用如下命令查询service属性

{% highlight shell %}
[root@node0 ~]# docker service inspect rng
...
"Mode": {
                "Replicated": {
                    "Replicas": 1
                }
            },
...
{% endhighlight %}

* 更新rng服务属性

{% highlight shell %}
[root@node0 ~]# docker service update rng --replicas 5
rng
[root@node0 ~]# docker service ls
ID            NAME      REPLICAS  IMAGE                                   COMMAND
0pcuhiw9jmu4  rng       5/5       localhost:5000/dockercoins_rng:v0.1     
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1  
3l07xdfj4rjk  redis     1/1       redis                                   
52c93icugean  webui     1/1       localhost:5000/dockercoins_webui:v0.1   
cgk5fahusjsy  registry  1/1       registry                                
clvgacmx24p7  worker    10/10     localhost:5000/dockercoins_worker:v0.1
{% endhighlight %}

![1]({{ site.url }}/pic/docker-swarm-3/67606a5b-57a8-4cda-8843-872d85faab0e.png)



可以看到每秒可以产生约30个docker币了.不过这离我们的理论值10个worker每秒产生40个docker币还差一点.

----------------------

## 找到程序瓶颈

现在我们整个环境如下:

10 个 worker,5 个rng,1 个 hasher,1 个 webui，1个redis

为了找到程序瓶颈, 我们可以启动一个临时容器, 对我们的rng和hasher服务做个简单的压力测试.

启动一个临时容器, 使用alpine镜像, 连接到dockercoins网络中.

{% highlight shell %}
[root@node0 ~]# docker service create --network dockercoins --name debug --mode global alpine sleep 1000000000
anm91nvl3sdm6wg24v1zokh02
{% endhighlight %}

\-\-mode globle：就是在swarm集群的每个节点创建一个容器副本

sleep 1000000000 ：想保持这个容器, 方便我们debug.

下面我们进入debug这个容器, 安装测试软件, 对我们的rng和hasher服务进行性能测试.

{% highlight shell %}
[root@node0 ~]# docker ps
CONTAINER ID        IMAGE                                    COMMAND                  CREATED             STATUS              PORTS               NAMES
70322a566b83        alpine:latest                            "sleep 1000000000"       9 seconds ago       Up 7 seconds                            debug.0.7llk0pcgzyv78mfg364tykbwh
bd97743256b5        localhost:5000/dockercoins_worker:v0.1   "python worker.py"       7 minutes ago       Up 6 minutes                            worker.4.15ynwltm1315xmf8nwzff16ht
e2f26f1ea927        localhost:5000/dockercoins_worker:v0.1   "python worker.py"       7 minutes ago       Up 6 minutes                            worker.7.a4bj2o3lc70e9jbo7uy7t6exy
73da7085cc92        registry:latest                          "/entrypoint.sh /etc/"   36 minutes ago      Up 36 minutes       5000/tcp            registry.1.0s4yz56rizgp7pi2e8vlk9t97
{% endhighlight %}


* 进入容器内部

{% highlight shell %}
[root@node0 ~]# docker exec -it 70322a566b83 sh
{% endhighlight %}

安装性能测试工具curl ab和drill

{% highlight shell %}
/ # apk add --update curl apache2-utils drill
{% endhighlight %}

## 负载均衡模式

* 检查rng服务

{% highlight shell %}
/ # drill rng
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 45767
;; flags: qr rd ra ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 
;; QUESTION SECTION:
;; rng. IN      A
;; ANSWER SECTION:
rng.    600     IN      A       10.0.0.6
;; AUTHORITY SECTION:
;; ADDITIONAL SECTION:
;; Query time: 0 msec
;; SERVER: 127.0.0.11
;; WHEN: Sun Nov 13 06:30:23 2016
;; MSG SIZE  rcvd: 40
/ # 
{% endhighlight %}

可以看到rng服务的IP地址是10.0.0.6, 可是我们一共有5个rng服务, 其实这个IP地址是swarm集群分配给所有rng服务负载均衡的VIP.

swarm集群负载均衡service有两种方式VIP和DNSRR:

* VIP模式每个service会得到一个 virtual IP地址作为服务请求的入口. 基于virtual IP进行负载均衡.
* DNSRR模式service利用DNS解析来进行负载均衡, 这种模式在旧的Docker Engine下, 经常行为诡异…所以不推荐.

如何查看service的负载均衡模式?

{% highlight shell %}
[root@node1 ~]# docker service inspect rng
...
            "EndpointSpec": {
                "Mode": "vip"
            }
        },
        "Endpoint": {
            "Spec": {
                "Mode": "vip"
            },
            "VirtualIPs": [
                {
                    "NetworkID": "8qvpdmja03td9dui3njmad822",
                    "Addr": "10.0.0.6/24"
                }
            ]
        },
        "UpdateStatus": {
            "StartedAt": "0001-01-01T00:00:00Z",
            "CompletedAt": "0001-01-01T00:00:00Z"
        }
    }
...
{% endhighlight %}

修改一个service的模式, 使用如下命令


{% highlight shell %}
docker service update --endpoint-mode [vip|dnssrr] Service-name
{% endhighlight %}

### rng服务压力测试

介绍完负载均衡模式，下面使用ab对rng服务进行简单的压力测试

测试之前关闭所有的worker服务，避免worker服务产生干扰

{% highlight shell %}
[root@node1 ~]# docker service scale worker=0
worker scaled to 0
[root@node1 ~]# docker service ls
ID            NAME      REPLICAS  IMAGE                                   COMMAND
0pcuhiw9jmu4  rng       5/5       localhost:5000/dockercoins_rng:v0.1     
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1  
3l07xdfj4rjk  redis     1/1       redis                                   
52c93icugean  webui     1/1       localhost:5000/dockercoins_webui:v0.1   
anm91nvl3sdm  debug     global    alpine                                  sleep 1000000000
cgk5fahusjsy  registry  1/1       registry                                
clvgacmx24p7  worker    0/0       localhost:5000/dockercoins_worker:v0.1  
{% endhighlight %}

模拟单个客户端,发送50个请求给rng服务

{% highlight shell %}
/ # ab -c 1 -n 50 http://rng/10
This is ApacheBench, Version 2.3 <$Revision: 1748469 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/
Benchmarking rng (be patient).....done
Server Software:        Werkzeug/0.11.11
Server Hostname:        rng
Server Port:            80
Document Path:          /10
Document Length:        10 bytes
Concurrency Level:      1
Time taken for tests:   5.383 seconds
Complete requests:      50
Failed requests:        0
Total transferred:      8250 bytes
HTML transferred:       500 bytes
Requests per second:    9.29 [#/sec] (mean)
Time per request:       107.655 [ms] (mean)
Time per request:       107.655 [ms] (mean, across all concurrent requests)
Transfer rate:          1.50 [Kbytes/sec] received
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    1   0.5      1       4
Processing:   104  106   1.7    105     111
Waiting:      104  105   1.4    105     110
Total:        105  107   1.8    107     113
Percentage of the requests served within a certain time (ms)
  50%    107
  66%    108
  75%    108
  80%    109
  90%    110
  95%    111
  98%    113
  99%    113
 100%    113 (longest request)
{% endhighlight %}

模拟50个并发客户端, 发送50个请求

{% highlight shell %}g
/ # ab -c 50 -n 50 http://rng/10
This is ApacheBench, Version 2.3 <$Revision: 1748469 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/
Benchmarking rng (be patient).....done
Server Software:        Werkzeug/0.11.11
Server Hostname:        rng
Server Port:            80
Document Path:          /10
Document Length:        10 bytes
Concurrency Level:      50
Time taken for tests:   1.127 seconds
Complete requests:      50
Failed requests:        0
Total transferred:      8250 bytes
HTML transferred:       500 bytes
Requests per second:    44.38 [#/sec] (mean)
Time per request:       1126.716 [ms] (mean)
Time per request:       22.534 [ms] (mean, across all concurrent requests)
Transfer rate:          7.15 [Kbytes/sec] received
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        9   18   5.5     19      31
Processing:   105  590 308.8    626    1096
Waiting:      104  590 309.1    625    1095
Total:        128  609 304.5    643    1105
Percentage of the requests served within a certain time (ms)
  50%    643
  66%    781
  75%    860
  80%    960
  90%   1061
  95%   1071
  98%   1105
  99%   1105
 100%   1105 (longest request)
{% endhighlight %}

可以看出单个客户端的时候rng的响应时间平均100+ms, 多并发情况下增加到大约1000ms+.

### hasher服务压力测试

hasher的服务测试稍微复杂点, 因为hasher服务需要POST一个随机的bytes数据.

所以我们需要先通过curl制作一个bytes数据文件

{% highlight shell %}
/ # curl http://rng/10 > /tmp/random
{% endhighlight %}

模拟单客户端,发送50个请求给hasher服务

{% highlight shell %}
/ # ab -c 1 -n 50 -T application/octet-stream -p /tmp/random http://hasher/
This is ApacheBench, Version 2.3 <$Revision: 1748469 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/
Benchmarking hasher (be patient).....done
Server Software:        thin
Server Hostname:        hasher
Server Port:            80
Document Path:          /
Document Length:        64 bytes
Concurrency Level:      1
Time taken for tests:   5.291 seconds
Complete requests:      50
Failed requests:        0
Total transferred:      10450 bytes
Total body sent:        7250
HTML transferred:       3200 bytes
Requests per second:    9.45 [#/sec] (mean)
Time per request:       105.820 [ms] (mean)
Time per request:       105.820 [ms] (mean, across all concurrent requests)
Transfer rate:          1.93 [Kbytes/sec] received
                        1.34 kb/s sent
                        3.27 kb/s total
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    1   0.4      1       3
Processing:   104  105   0.9    104     107
Waiting:      103  104   0.9    104     107
Total:        104  106   1.1    105     108
WARNING: The median and mean for the processing time are not within a normal deviation
        These results are probably not that reliable.
Percentage of the requests served within a certain time (ms)
  50%    105
  66%    106
  75%    106
  80%    106
  90%    108
  95%    108
  98%    108
  99%    108
 100%    108 (longest request)
{% endhighlight %}

模拟50个并发客户端, 发送50个请求

{% highlight shell %}
/ # ab -c 50 -n 50 -T application/octet-stream -p /tmp/random http://hasher/
This is ApacheBench, Version 2.3 <$Revision: 1748469 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/
Benchmarking hasher (be patient).....done
Server Software:        thin
Server Hostname:        hasher
Server Port:            80
Document Path:          /
Document Length:        64 bytes
Concurrency Level:      50
Time taken for tests:   0.347 seconds
Complete requests:      50
Failed requests:        0
Total transferred:      10450 bytes
Total body sent:        7250
HTML transferred:       3200 bytes
Requests per second:    143.93 [#/sec] (mean)
Time per request:       347.381 [ms] (mean)
Time per request:       6.948 [ms] (mean, across all concurrent requests)
Transfer rate:          29.38 [Kbytes/sec] received
                        20.38 kb/s sent
                        49.76 kb/s total
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        5   11   5.0     11      19
Processing:   140  210  66.7    225     323
Waiting:      137  209  67.8    224     323
Total:        155  222  62.3    234     329
Percentage of the requests served within a certain time (ms)
  50%    234
  66%    236
  75%    236
  80%    316
  90%    326
  95%    328
  98%    329
  99%    329
 100%    329 (longest request)
{% endhighlight %}

从结果可以看出, 单客户端hasher平均响应时间100+ms, 50并发平均响应时间300+ms.

### 程序瓶颈分析


也许你注意到了, 单客户端请求的测试rng平均响应时间约100+ms,hasher的响应时间100+ms.

我们搭了swarm集群, Scale了应用.为啥单个请求的时间都在100ms以上呢.

当你做了很多的优化, 想了很多可能, 应用的性能还是上不去, 这时候一般都是开发挖的坑!!!!

下一节 我们会去看看rng和hasher的代码, 找到响应时间在100ms的原因. 更新代码, 试图缩短响应时间.当然这个系列是介绍docker swarm的. 所以下面章节重点是介绍我们的应用程序更新以后——如何使用swarm集群roll update容器服务.

---------------

end

