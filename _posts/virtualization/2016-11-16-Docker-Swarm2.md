---
layout: post
title: "Docker Swarm 2 -- 初试swarm集群"
categories: Docker
---

* content
{:toc}

## 前言

为了解决上一章的docker币应用单节点瓶颈问题，我们引出以下章节内容：

1. 部署docker swarm集群，worker节点分布到不同虚拟机上运行

2. scale out rng节点，横向扩展多个rng节点消除瓶颈

## Swarm模式简介

Docker Engine 1.12 集成了Swarm集群工具;主要使用四个命令工具管理swarm集群

* docker swarm：开启swarm模式；加入swarm集群；配置集群参数

* docker node：查询节点信息；提升/降低管理节点；管理swarm节点主机

* docker network：创建管理集群网络

* docker service：创建管理集群服务

-------------------

## 创建Swarm集群

在node0上初始化swarm集群

> Note：只需在node0上初始化swarm集群，其他的node只需加入这个集群就行

{% highlight shell %}
[root@node0 ~]# docker swarm init --advertise-addr 192.168.1.130
Swarm initialized: current node (7gf4hu3kyc5u0vbyq0k0p2xoa) is now a manager.
To add a worker to this swarm, run the following command:
    docker swarm join \
    --token SWMTKN-1-2wfd88l7jst8tomvk72z9k3byfatfprdtbwxnwq917tumprxua-6t5xnckb1d8ylquqqpylx65m7 \
    192.168.1.130:2377
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
{% endhighlight %}

\-\-advertise-addr参数, 后面跟你swarm集群的通讯地址, 也就是node0的IP地址.

查询加入swarm集群的命令和密钥可以使用如下方式分别查看worker节点和manager节点的加入命令

{% highlight shell %}
[root@node0 ~]# docker swarm join-token worker
[root@node0 ~]# docker swarm join-token manager
{% endhighlight %}

其他节点服务器,以worker角色加入swarm集群：

{% highlight shell %}
docker swarm join \
    --token SWMTKN-1-2wfd88l7jst8tomvk72z9k3byfatfprdtbwxnwq917tumprxua-6t5xnckb1d8ylquqqpylx65m7 \
    192.168.1.130:2377
{% endhighlight %}

其他节点服务器,以manager角色加入swarm集群：

{% highlight shell %}
docker swarm join \
    --token SWMTKN-1-2wfd88l7jst8tomvk72z9k3byfatfprdtbwxnwq917tumprxua-483ps7pt4xcroqjxuekwnt2ra \
    192.168.1.130:2377
{% endhighlight %}


检查node0 docker swarm mode信息:

{% highlight shell %}
[root@node0 ~]# docker info
...
Swarm: active
 NodeID: 7gf4hu3kyc5u0vbyq0k0p2xoa
 Is Manager: true
 ClusterID: do10yu1fqms1foc2bqu2y4upm
 Managers: 1
 Nodes: 1
 Orchestration:
  Task History Retention Limit: 5
 Raft:
  Snapshot Interval: 10000
  Heartbeat Tick: 1
  Election Tick: 3
 Dispatcher:
  Heartbeat Period: 5 seconds
 CA Configuration:
  Expiry Duration: 3 months
 Node Address: 192.168.1.130
...
{% endhighlight %}

查看swarm集群node列表

{% highlight shell %}
[root@node0 ~]# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
7gf4hu3kyc5u0vbyq0k0p2xoa *  node0     Ready   Active        Leader
{% endhighlight %}

目前swarm集群中只有一个节点，我们先把其他所有节点都以worker身份加入到我们的集群中

在node0 上以ssh方式批量将node1-node4加入集群

{% highlight shell %}
for N in $(seq 1 4); \
do ssh node${N} \
docker swarm join \
--token SWMTKN-1-2wfd88l7jst8tomvk72z9k3byfatfprdtbwxnwq917tumprxua-6t5xnckb1d8ylquqqpylx65m7 \
192.168.1.130:2377 \
;done
{% endhighlight %}

验证集群节点列表，可以看到所有的节点服务器都加入到了swarm集群了

{% highlight shell %}
[root@node0 ~]# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
23gu2oc96su1tn7opoapv1ikd    node4     Ready   Active        
57xdzm22a9syigrnbg8fqqox6    node3     Ready   Active        
7gf4hu3kyc5u0vbyq0k0p2xoa *  node0     Ready   Active        Leader
8ko8h0egr0vgydoww6poysg70    node1     Ready   Active        
8ls5ajugjza3utgrwy2qhgxe6    node2     Ready   Active        
{% endhighlight %}

不过我们的集群只有一个manager节点node0，为了swarm集群的高可用，我们再建立2个manager节点

只需通过命令提升node1 和node2 为manager节点

{% highlight shell %}
[root@node0 ~]# docker node promote node1 node2
Node node1 promoted to a manager in the swarm.
Node node2 promoted to a manager in the swarm.
{% endhighlight %}

查看集群节点信息

{% highlight shell %}
[root@node0 ~]# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
23gu2oc96su1tn7opoapv1ikd    node4     Ready   Active        
57xdzm22a9syigrnbg8fqqox6    node3     Ready   Active        
7gf4hu3kyc5u0vbyq0k0p2xoa *  node0     Ready   Active        Leader
8ko8h0egr0vgydoww6poysg70    node1     Ready   Active        Reachable
8ls5ajugjza3utgrwy2qhgxe6    node2     Ready   Active        Reachable
{% endhighlight %}

我们看到：现在有两个manager节点和一个leader节点，现在你也可以在node1 node2上管理整个集群了

{% highlight shell %}
[root@node0 ~]# ssh node1 docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
23gu2oc96su1tn7opoapv1ikd    node4     Ready   Active        
57xdzm22a9syigrnbg8fqqox6    node3     Ready   Active        
7gf4hu3kyc5u0vbyq0k0p2xoa    node0     Ready   Active        Leader
8ko8h0egr0vgydoww6poysg70 *  node1     Ready   Active        Reachable
8ls5ajugjza3utgrwy2qhgxe6    node2     Ready   Active        Reachable
[root@node0 ~]# ssh node2 docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
23gu2oc96su1tn7opoapv1ikd    node4     Ready   Active        
57xdzm22a9syigrnbg8fqqox6    node3     Ready   Active        
7gf4hu3kyc5u0vbyq0k0p2xoa    node0     Ready   Active        Leader
8ko8h0egr0vgydoww6poysg70    node1     Ready   Active        Reachable
8ls5ajugjza3utgrwy2qhgxe6 *  node2     Ready   Active        Reachable
{% endhighlight %}

到此为止，swarm集群搭建完毕

------------------------

## 在Swarm集群上运行服务

使用docker service 命令创建集群服务

例如：在swarm集群上启动一个alping镜像ping 8.8.8.8

{% highlight shell %}
[root@node0 ~]# docker service create --name ping-google alpine ping 8.8.8.8
dnnpgwqjlnixm0xlzi9sggzr2
{% endhighlight %}

查看我们刚刚创建的service

{% highlight shell %}
[root@node0 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE   COMMAND
dnnpgwqjlnix  ping-google  1/1       alpine  ping 8.8.8.8
{% endhighlight %}

查看服务跑在哪台节点服务器上

{% highlight shell %}
[root@node0 ~]# docker service ps ping-google
ID                         NAME           IMAGE   NODE   DESIRED STATE  CURRENT STATE               ERROR
3j9l5pfug5p6wmj0gw8wwizwc  ping-google.1  alpine  node0  Running        Running about a minute ago  
{% endhighlight %}

当前service是跑在node0上，当前状态是运行状态

> 如果容器启动失败，swarm集群会不断的重启容器，直到成功为止

-------------------------------

## Scale out服务

我们scale服务到10个副本

{% highlight shell %}
[root@node0 ~]# docker service scale ping-google=10
ping-google scaled to 10
[root@node0 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE   COMMAND
dnnpgwqjlnix  ping-google  1/10      alpine  ping 8.8.8.8
{% endhighlight %}

注意REPLICAS现在显示1/10表示这个service一共有10个副本,现在成功运行了1个. 集群正在启动其他的副本.

这这时候查看service进程

{% highlight shell %}
[root@node0 ~]# docker service ps ping-google
ID                         NAME            IMAGE   NODE   DESIRED STATE  CURRENT STATE           ERROR
3j9l5pfug5p6wmj0gw8wwizwc  ping-google.1   alpine  node0  Running        Running 5 minutes ago   
8dg45mx9xdnpc5skr6cf3qg3h  ping-google.2   alpine  node1  Running        Running 35 seconds ago  
29mphyj1vskqfgza443m5rjzu  ping-google.3   alpine  node3  Running        Running 16 seconds ago  
bozqod88a84c886lnrn9nyix8  ping-google.4   alpine  node3  Running        Running 28 seconds ago  
1hlyg2qlu1memo64yowqsla1z  ping-google.5   alpine  node2  Running        Running 25 seconds ago  
685ltkj4l3lfxp1p9horqn05q  ping-google.6   alpine  node1  Running        Running 35 seconds ago  
60bb0vgik71xqpgprgz6axqu3  ping-google.7   alpine  node0  Running        Running 21 seconds ago  
8qud2w1k0y6c9jey9at6mcfoe  ping-google.8   alpine  node4  Running        Running 32 seconds ago  
c0g1lqc6ukzacz5qhvftok4od  ping-google.9   alpine  node4  Running        Running 19 seconds ago  
04zq57a07b3vxpp1d1b03awd8  ping-google.10  alpine  node2  Running        Running 25 seconds ago  
[root@node0 ~]# docker service ls
ID            NAME         REPLICAS  IMAGE   COMMAND
dnnpgwqjlnix  ping-google  10/10     alpine  ping 8.8.8.8
{% endhighlight %}

所有副本都启动成功了，swarm集群会自动编排10个副本在我们的5台虚拟机上，每个节点会启动2个容器

{% highlight shell %}
[root@node0 ~]# ssh node0 docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
e19530521f15        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            ping-google.7.60bb0vgik71xqpgprgz6axqu3
d08f19033a61        alpine:latest       "ping 8.8.8.8"      7 minutes ago       Up 7 minutes                            ping-google.1.3j9l5pfug5p6wmj0gw8wwizwc
[root@node0 ~]# ssh node1 docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
1df631edd4e3        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            ping-google.6.685ltkj4l3lfxp1p9horqn05q
d92107463664        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            ping-google.2.8dg45mx9xdnpc5skr6cf3qg3h
[root@node0 ~]# ssh node2 docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
f8ca000b213d        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            ping-google.10.04zq57a07b3vxpp1d1b03awd8
2c9d2eed1223        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            ping-google.5.1hlyg2qlu1memo64yowqsla1z
[root@node0 ~]# ssh node3 docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
3cfc4404460f        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            ping-google.3.29mphyj1vskqfgza443m5rjzu
1913a46d8885        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            ping-google.4.bozqod88a84c886lnrn9nyix8
[root@node0 ~]# ssh node4 docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
5112d6ea366e        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            ping-google.9.c0g1lqc6ukzacz5qhvftok4od
9af36d32f162        alpine:latest       "ping 8.8.8.8"      2 minutes ago       Up 2 minutes                            ping-google.8.8qud2w1k0y6c9jey9at6mcfoe
{% endhighlight %}

----------------

## 暴露一个服务端口

和我们单机使用docker一样，你可以将容器的端口暴露到主机网络中，swarm集群的service端口暴露端口有如下特性:

* 公共的端口会暴露在每一个swarm集群中的节点服务器上.
* 请求进如公共端口后会负载均衡到所有的sevice实例上.

使用命令：docker service create -p

下面我们启动10个Nginx服务，发布端口9999

{% highlight shell %}
[root@node0 ~]# docker service create --name my_nginx -p 9999:80 --replicas 10 nginx
{% endhighlight %}

使用以下命令监控 service 创建过程

{% highlight shell %}
[root@node0 ~]# watch docker service ps my_nginx
{% endhighlight %}

注意CURRENT STATE 列, 一个service副本的创建过程,会经历以下几个状态:

* accepted：任务已经被分配到某一个节点执行
* preparing：准备资源, 现在来说一般是从网络拉取image
* running：副本运行成功
* shutdown：报错,被停止了

> 当一个任务被终止stoped or killed.., 任务不能被重启, 但是一个替代的任务会被创建.

在swarm集群任意节点上都可访问Nginx服务；验证如下：

{% highlight shell %}
[root@node0 ~]# curl localhost:9999
{% endhighlight %}

----------------------------------

ok，docker swarm集群已经搭建并且测试完毕；下一章在swarm集群上部署我们的docker币应用

## 清理环境

清除所有servcie

{% highlight shell %}
[root@node0 ~]# docker service ls -q|xargs docker service rm
{% endhighlight %}

清除指定servcie

{% highlight shell %}
docker service rm ping-google
{% endhighlight %}


----------------

20161222 ADD：

此节点不运行任务

> docker node update –availability drain node01

