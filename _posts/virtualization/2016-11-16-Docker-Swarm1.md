---
layout: post
title: "Docker Swarm 1 -- 引入swarm"
categories: Docker
---

* content
{:toc}

## 前言

自从dockercon 2016 发布docker1.12版本以来，经历了几个RC版本，docker1.12终于迎来了第一个稳定版；

docker1.12展示了docker大统一平台的野心，集成了swarmkit，不需要安装额外的软件包，使用简单的命令启动创建docker swarm集群；

集成了swarm集群的安全特性，集成了K-V存储，不需要额外部署etcd或consul；

本系列教程采用docker币应用程序模拟生产应用程序会遇到的问题;

## 本文参考

* [docker-swarm-in-action-01](https://guai.im/2016/08/01/docker-swarm-in-action-01/)
* [docker-swarm-in-action-02](https://guai.im/2016/08/01/docker-swarm-in-action-01/)
* [docker-swarm-in-action-03](https://guai.im/2016/08/01/docker-swarm-in-action-01/)
* [docker-swarm-in-action-04](https://guai.im/2016/08/01/docker-swarm-in-action-01/)
* [docker-swarm-in-action-05](https://guai.im/2016/08/01/docker-swarm-in-action-01/)
* [docker-swarm-in-action-06](https://guai.im/2016/08/01/docker-swarm-in-action-01/)

---------------

## 实验环境准备

使用VMware创建5台虚拟机搭建docker swarm集群：

* 系统要求：CentOS-7.2
* 内核要求：3.10.0-327.el7.x86_64
* Docker版本：Docker version 1.12.3, build 6b644ec

主机名和IP地址规划如下：

node0 ==== 192.168.1.130

node1 ==== 192.168.1.131

node2 ==== 192.168.1.132

node3 ==== 192.168.1.133

node4 ==== 192.168.1.134

----------------------

## 系统环境准备

* 配置node0到所有节点密钥登陆（非必要）
* 配置hosts列表用于主机名-IP的解析（必要）
* 安装ntp服务用于同步时间（必要）

### ssh-key密钥登录

{% highlight shell %}
[root@node0 ~]# ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ''""
[root@node0 ~]# ssh-copy-id 192.168.1.130
The authenticity of host 'node0 (192.168.1.130)' can't be established.
ECDSA key fingerprint is 54:ef:99:03:3d:0d:e5:09:73:f6:8d:f2:a6:a2:29:fc.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@node0's password: 输入192.168.1.130的root密码
Number of key(s) added: 1
Now try logging into the machine, with:   "ssh 'node0'"
and check to make sure that only the key(s) you wanted were added.

[root@node0 ~]# ssh-copy-id 192.168.1.131
...
[root@node0 ~]# ssh-copy-id 192.168.1.132
...
[root@node0 ~]# ssh-copy-id 192.168.1.133
...
[root@node0 ~]# ssh-copy-id 192.168.1.134
{% endhighlight %}

### 修改hosts文件

{% highlight shell %}
[root@node0 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.130   node0
192.168.1.131   node1
192.168.1.132   node2
192.168.1.133   node3
192.168.1.134   node4
{% endhighlight %}

### 测试hosts文件解析与密钥登陆

{% highlight shell %}
[root@node0 ~]# for N in $(seq 0 4);do ssh node${N} hostname;done
node0
node1
node2
node3
node4
{% endhighlight %}

### 批量修改hosts文件

{% highlight shell %}
[root@node0 ~]# for N in $(seq 0 4);do scp /etc/hosts node${N}:/etc/hosts;done
{% endhighlight %}


### 安装配置NTP时间服务

以下命令在所有node节点都要执行

{% highlight shell %}
yum install ntp -y
systemctl start ntpd
systemctl enable ntpd
touch /var/spool/cron/root
echo "*/5 * * * * /usr/sbin/ntpdate ntp1.aliyun.com >/dev/null 2>&1" >>/var/spool/cron/root
{% endhighlight %}

-----------------------

## 安装docker 1.12

所有node节点都需安装docker

{% highlight shell %}
下载RPM安装包
wget https://yum.dockerproject.org/repo/main/centos/7/Packages/docker-engine-1.12.3-1.el7.centos.x86_64.rpm
使用yum localinstall安装自动解决依赖关系
yum localinstall -y docker-engine-1.12.3-1.el7.centos.x86_64.rpm --nogpgcheck
启动docker并设置开机自启动
systemctl start docker
ystemctl enable docker
验证docker版本
docker -v
Docker version 1.12.3, build 6b644ec
下载docker-compose工具
curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose -v
docker-compose version 1.8.0, build f3628c7
{% endhighlight %}

------------------

## 运行docker币应用

### 下载docker币

在node0上执行以下命令：

{% highlight shell %}
安装git
[root@node0 ~]# yum install git -y
克隆git 库
[root@node0 ~]# git clone https://github.com/jpetazzo/orchestration-workshop.git
进入dockercoins应用目录
[root@node0 ~]# cd orchestration-workshop/dockercoins
[root@node0 dockercoins]# ls
docker-compose.yml             docker-compose.yml-portmap     hasher     webui
docker-compose.yml-ambassador  docker-compose.yml-scaled-rng  ports.yml  worker
docker-compose.yml-logging     docker-compose.yml-v2          rng
{% endhighlight %}

> 该github库中包含了, workshop的详细文档和配置. 部署脚本, PPT等.有基础的同学可以自己看看.

### docker币应用架构组成

{% highlight shell %}
[root@node0 dockercoins]# cat docker-compose.yml
version: "2"
services:
  rng:
    build: rng
    ports:
    - "8001:80"
  hasher:
    build: hasher
    ports:
    - "8002:80"
  webui:
    build: webui
    ports:
    - "8000:80"
    volumes:
    - "./webui/files/:/files/"
  redis:
    image: redis
  worker:
    build: worker
{% endhighlight %}

docker币一共对应4个服务：rng，hasher，webui，worker；

详情可进入对应目录查看Dockerfile描述文件

* rng：是一个web service 生成随机的bytes数据.
* hasher：是一个计算POST 数据hash值的web service.
* worker：工作节点；后台调用rng 生成随机bytes数据, 然后将数据post到hasher服务计算hash值, 如果hashe值开头为0.则产生一个docker币.并将docker币保存到redis数据库中.
* webui：一个web接口用来监控整个系统，去读redis数据库中的docker币数量并展示.

### 启动docker币引用

在node0上自动运行docker币应用

{% highlight shell %}
[root@node0 ~]# docker-compose -f /root/orchestration-workshop/dockercoins/docker-compose.yml up -d
{% endhighlight %}

> 由于某些特殊原因docker hub极不稳定，所以需要很长时间来下载镜像（可选择下面手动方式）

手动pull基础镜像

{% highlight shell %}
[root@node0 ~]# docker pull python:alpine
[root@node0 ~]# docker pull ruby:alpine
[root@node0 ~]# docker pull redis
[root@node0 ~]# docker pull node:4-slim
[root@node0 ~]# docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
node                 4-slim              b2bd4c48b5dc        3 days ago          205.3 MB
redis                latest              5f515359c7f8        4 days ago          182.9 MB
ruby                 alpine              35562b355c05        11 days ago         126.7 MB
python               alpine              8dd7712cca84        12 days ago         88.49 MB
{% endhighlight %}


手动Build docker币应用镜像

{% highlight shell %}
[root@node0 ~]# docker-compose -f /root/orchestration-workshop/dockercoins/docker-compose.yml build
...
根据每个应用目录下的Dockerfile build应用镜像, 需要一会儿....
build完成后, 你可以使用 docker image命令查看镜像

[root@node0 ~]# docker images
REPOSITORY           TAG                 IMAGE ID            CREATED              SIZE
dockercoins_webui    latest              c2d09bb0c442        About a minute ago   212.2 MB
dockercoins_rng      latest              ff1d886f2fa3        2 minutes ago        83.53 MB
dockercoins_hasher   latest              9d361dbaf589        2 minutes ago        310.6 MB
dockercoins_worker   latest              bef5d2dc4bd0        8 minutes ago        80.5 MB
...
{% endhighlight %}

手动启动docker币应用

{% highlight shell %}
[root@node0 ~]# docker-compose -f /root/orchestration-workshop/dockercoins/docker-compose.yml up
{% endhighlight %}

启动后可以看见应用日志滚动

使用Ctrl+C停止应用，使用-d参数后台启动docker币

{% highlight shell %}
[root@node0 ~]# docker-compose -f /root/orchestration-workshop/dockercoins/docker-compose.yml up -d
{% endhighlight %}

查看运行情况

{% highlight shell %}
[root@node0 ~]# docker-compose -f /root/orchestration-workshop/dockercoins/docker-compose.yml ps
        Name                      Command               State          Ports         
------------------------------------------------------------------------------------
dockercoins_hasher_1   ruby hasher.rb                   Up      0.0.0.0:8002->80/tcp 
dockercoins_redis_1    docker-entrypoint.sh redis ...   Up      6379/tcp             
dockercoins_rng_1      python rng.py                    Up      0.0.0.0:8001->80/tcp 
dockercoins_webui_1    node webui.js                    Up      0.0.0.0:8000->80/tcp 
dockercoins_worker_1   python worker.py                 Up                           
{% endhighlight %}

查看应用日志

{% highlight shell %}
[root@node0 ~]# docker-compose -f /root/orchestration-workshop/dockercoins/docker-compose.yml logs
{% endhighlight %}

滚动输出应用日志，每个容器输出最新10行

{% highlight shell %}
[root@node0 ~]# docker-compose -f /root/orchestration-workshop/dockercoins/docker-compose.yml logs --tail 10 --follow
{% endhighlight %}

通过浏览器访问http://192.168.1.130:8000 可以看到应用挖掘到的docker币情况

![1]({{ site.url }}/pic/docker-swarm-1/69cc3a57-1d97-4f13-91aa-c82f4d2defb9.png)

从webui可以开出我们的docker币应用每秒产生4个docker币

---------------------

## Scale up向上扩展应用

WOW~~我们的docker币应用已经正常工作了，每秒产生4个docker币

但是如果我们想产生更多的docker币怎么办？

前面已经介绍docker币的应用架构，worker服务负则产生docker币，所以我们尝试scale up worker服务

{% highlight shell %}
切换到dockerconins目录下运行docker-compose命令，即可不用指定yaml文件位置
[root@node0 dockercoins]# docker-compose scale worker=2
Creating and starting dockercoins_worker_2 ... done
[root@node0 dockercoins]# docker-compose ps
        Name                      Command               State          Ports         
------------------------------------------------------------------------------------
dockercoins_hasher_1   ruby hasher.rb                   Up      0.0.0.0:8002->80/tcp 
dockercoins_redis_1    docker-entrypoint.sh redis ...   Up      6379/tcp             
dockercoins_rng_1      python rng.py                    Up      0.0.0.0:8001->80/tcp 
dockercoins_webui_1    node webui.js                    Up      0.0.0.0:8000->80/tcp 
dockercoins_worker_1   python worker.py                 Up                           
dockercoins_worker_2   python worker.py                 Up                     
{% endhighlight %}

现在我们有2个worker工作了

查看webui：

![1]({{ site.url }}/pic/docker-swarm-1/09f14d56-ad13-49c0-9015-c6c92e24f378.png)

从webui可以看出我们的docker币应用每秒产生约8个docker币

WoW~这么简单，让我们来挖掘更多的docker币吧

{% highlight shell %}
[root@node0 dockercoins]# docker-compose scale worker=10
Creating and starting dockercoins_worker_3 ... done
Creating and starting dockercoins_worker_4 ... done
Creating and starting dockercoins_worker_5 ... done
Creating and starting dockercoins_worker_6 ... done
Creating and starting dockercoins_worker_7 ... done
Creating and starting dockercoins_worker_8 ... done
Creating and starting dockercoins_worker_9 ... done
Creating and starting dockercoins_worker_10 ... done
[root@node0 dockercoins]# docker-compose ps
        Name                       Command               State          Ports         
-------------------------------------------------------------------------------------
dockercoins_hasher_1    ruby hasher.rb                   Up      0.0.0.0:8002->80/tcp 
dockercoins_redis_1     docker-entrypoint.sh redis ...   Up      6379/tcp             
dockercoins_rng_1       python rng.py                    Up      0.0.0.0:8001->80/tcp 
dockercoins_webui_1     node webui.js                    Up      0.0.0.0:8000->80/tcp 
dockercoins_worker_1    python worker.py                 Up                           
dockercoins_worker_10   python worker.py                 Up                           
dockercoins_worker_2    python worker.py                 Up                           
dockercoins_worker_3    python worker.py                 Up                           
dockercoins_worker_4    python worker.py                 Up                           
dockercoins_worker_5    python worker.py                 Up                           
dockercoins_worker_6    python worker.py                 Up                           
dockercoins_worker_7    python worker.py                 Up                           
dockercoins_worker_8    python worker.py                 Up                           
dockercoins_worker_9    python worker.py                 Up             
{% endhighlight %}

我们现在有10个worker节点了，理论上我们应该每秒产生40个docker币

![1]({{ site.url }}/pic/docker-swarm-1/863297e6-afe8-4318-99ae-ce3373276910.png)

> 注意!!!：为什么现在每秒才产生约10个docker币???

----------------------------

## 程序的瓶颈

目前情况：

1个worker节点每秒产生4个docker币

2个worker节点每秒产生8个docker币

10个worker节点每秒产生10个docker币

Why？

我们所有的应用都跑在1台服务器上node0；10个worker造成服务器资源竞争；所以增加worker节点也不能改善性能；

我们测试一下rng shaher节点的负载，因为我们有10个worker节点，只有一个rng，hasher节点，所以瓶颈可能发生在rng或hasher节点

使用httping工具测试rng，hasher延迟

> Note：httping可以带端口去ping

{% highlight shell %}
[root@node0 dockercoins]# yum install epel-release -y
[root@node0 dockercoins]# yum install httping -y
{% endhighlight %}

测试rng延迟, docker-compose ps命令可以看到rng应用映射到本地8001端口

{% highlight shell %}
[root@node0 dockercoins]# httping -c 10 localhost:8001
PING localhost:8001 (/):
connected to 127.0.0.1:8001 (159 bytes), seq=0 time=783.34 ms 
connected to 127.0.0.1:8001 (159 bytes), seq=1 time=801.78 ms 
connected to 127.0.0.1:8001 (159 bytes), seq=2 time=832.84 ms 
connected to 127.0.0.1:8001 (159 bytes), seq=3 time=797.79 ms 
connected to 127.0.0.1:8001 (159 bytes), seq=4 time=818.90 ms 
connected to 127.0.0.1:8001 (159 bytes), seq=5 time=747.49 ms 
connected to 127.0.0.1:8001 (159 bytes), seq=6 time=780.67 ms 
connected to 127.0.0.1:8001 (159 bytes), seq=7 time=875.05 ms 
connected to 127.0.0.1:8001 (159 bytes), seq=8 time=825.70 ms 
connected to 127.0.0.1:8001 (159 bytes), seq=9 time=836.49 ms 
--- http://localhost:8001/ ping statistics ---
10 connects, 10 ok, 0.00% failed, time 18119ms
round-trip min/avg/max = 747.5/810.0/875.0 ms
{% endhighlight %}

测试hasher延迟, 8002端口

{% highlight shell %}
[root@node0 dockercoins]# httping -c 10 localhost:8002
PING localhost:8002 (/):
connected to 127.0.0.1:8002 (210 bytes), seq=0 time=  6.16 ms 
connected to 127.0.0.1:8002 (210 bytes), seq=1 time=  3.42 ms 
connected to 127.0.0.1:8002 (210 bytes), seq=2 time=  5.52 ms 
connected to 127.0.0.1:8002 (210 bytes), seq=3 time=  2.31 ms 
connected to 127.0.0.1:8002 (210 bytes), seq=4 time=  6.20 ms 
connected to 127.0.0.1:8002 (210 bytes), seq=5 time=  2.55 ms 
connected to 127.0.0.1:8002 (210 bytes), seq=6 time=  4.24 ms 
connected to 127.0.0.1:8002 (210 bytes), seq=7 time=  2.80 ms 
connected to 127.0.0.1:8002 (210 bytes), seq=8 time=  9.32 ms 
connected to 127.0.0.1:8002 (210 bytes), seq=9 time=  4.07 ms 
--- http://localhost:8002/ ping statistics ---
10 connects, 10 ok, 0.00% failed, time 10069ms
round-trip min/avg/max = 2.3/4.7/9.3 ms
{% endhighlight %}

从以上结果可以看出，性能瓶颈应该在rng节点

## 如何解决瓶颈

建立docker集群，worker节点分布到不同虚拟机上运行

scale out rng节点，横向扩展多个rng节点消除瓶颈

> 由此引出docker swarm集群

------------------------

## 清除当前环境

{% highlight shell %}
[root@node0 dockercoins]# docker-compose down
Stopping dockercoins_worker_9 ... done
Stopping dockercoins_worker_6 ... done
Stopping dockercoins_worker_10 ... done
Stopping dockercoins_worker_4 ... done
Stopping dockercoins_worker_5 ... done
Stopping dockercoins_worker_7 ... done
Stopping dockercoins_worker_8 ... done
Stopping dockercoins_worker_3 ... done
Stopping dockercoins_worker_2 ... done
Stopping dockercoins_webui_1 ... done
Stopping dockercoins_hasher_1 ... done
Stopping dockercoins_rng_1 ... done
Stopping dockercoins_worker_1 ... done
Stopping dockercoins_redis_1 ... done
Removing dockercoins_worker_9 ... done
Removing dockercoins_worker_6 ... done
Removing dockercoins_worker_10 ... done
Removing dockercoins_worker_4 ... done
Removing dockercoins_worker_5 ... done
Removing dockercoins_worker_7 ... done
Removing dockercoins_worker_8 ... done
Removing dockercoins_worker_3 ... done
Removing dockercoins_worker_2 ... done
Removing dockercoins_webui_1 ... done
Removing dockercoins_hasher_1 ... done
Removing dockercoins_rng_1 ... done
Removing dockercoins_worker_1 ... done
Removing dockercoins_redis_1 ... done
Removing network dockercoins_default
[root@node0 dockercoins]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS 
{% endhighlight %}

----------------------

end


