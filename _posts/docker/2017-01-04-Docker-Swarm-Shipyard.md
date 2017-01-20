---
layout: post
title: "Docker Swarm Shipyard"
categories: Docker
---


* content
{:toc}

## Reference

> https://www.shipyard-project.com/docs/

> https://github.com/shipyard/shipyard

## Introduction

Shipard是docker集群的可视化解决方案，shipyard利用内置的swarm组建管理集群，并提供图形化管理界面。可管理容器，镜像，节点，私有仓库等。还提供了用户认证和访问控制。

Shipyard完全100%利用的`Docker Remote API`实现的

---------

## Shipyard Deployment

### Automated Deployment

> curl -sSL https://shipyard-project.com/deploy \| bash -s

### Manual Deployment

Datastore

{% highlight shell %}
docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-rethinkdb \
    rethinkdb
{% endhighlight %}

Discovery

{% highlight shell %}
docker run \
    -ti \
    -d \
    -p 4001:4001 \
    -p 7001:7001 \
    --restart=always \
    --name shipyard-discovery \
    microbox/etcd -name discovery
{% endhighlight %}

Proxy

{% highlight shell %}
docker run \
    -ti \
    -d \
    -p 2375:2375 \
    --hostname=$HOSTNAME \
    --restart=always \
    --name shipyard-proxy \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -e PORT=2375 \
    shipyard/docker-proxy:latest
{% endhighlight %}


Swarm Manager

{% highlight shell %}
docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-swarm-manager \
    swarm:latest \
    manage --host tcp://0.0.0.0:3375 etcd://<IP-OF-HOST>:4001
{% endhighlight %}

Swarm Agent


{% highlight shell %}
docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-swarm-agent \
    swarm:latest \
    join --addr <ip-of-host>:2375 etcd://<ip-of-host>:4001
{% endhighlight %}

Controller

{% highlight shell %}
docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-controller \
    --link shipyard-rethinkdb:rethinkdb \
    --link shipyard-swarm-manager:swarm \
    -p 8080:8080 \
    shipyard/shipyard:latest \
    server \
    -d tcp://swarm:3375
{% endhighlight %}


--------------------

## Compile Shipyard

### go

{% highlight shell %}
wget https://storage.googleapis.com/golang/go1.7.4.linux-amd64.tar.gz
tar -C /usr/local/ -xf go1.7.4.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version
{% endhighlight %}

### godep



{% highlight shell %}
export GOPATH=/usr/
go get github.com/tools/godep
{% endhighlight %}


### nvm

{% highlight shell %}
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.31.1/install.sh | bash
source ~/.bashrc
nvm --version
{% endhighlight %}



### nodejs && npm


{% highlight shell %}
yum install nodejs
node -v
npm -v
{% endhighlight %}


### bower

{% highlight shell %}
npm install -g bower --registry=http://registry.npm.taobao.org
bower -v
{% endhighlight %}

### git clone

{% highlight shell %}
git clone https://github.com/shipyard/shipyard.git
mkdir -p /usr/local/go/src/github.com/shipyard/
mv shipyard /usr/local/go/src/github.com/shipyard/
{% endhighlight %}



### modify index.html


{% highlight shell %}
vim shipyard/controller/static/app/login/login.html

 4 <h1 style="margin-top: 100px; margin-bottom: 50px; font-family: 'Poiret One', sans-serif; font-size: 72px; color: #ffffff;">Mimte-Docker</h1>
{% endhighlight %}


{% highlight shell %}
cd /usr/local/go/src/github.com/shipyard/shipyard
make build
make media
{% endhighlight %}


{% highlight shell %}
cd /usr/local/go/src/github.com/shipyard/shipyard/controller
docker build -t shipyard/shipyard:v1 ./
Sending build context to Docker daemon 38.34 MB
Step 1 : FROM alpine:latest
 ---> 0766572b4bac
Step 2 : RUN apk add --update git ca-certificates &&     rm -rf /var/cache/apk/*
 ---> Running in 87b47f8d5d9b
fetch http://dl-cdn.alpinelinux.org/alpine/v3.4/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.4/community/x86_64/APKINDEX.tar.gz
(1/6) Installing ca-certificates (20160104-r4)
(2/6) Installing libssh2 (1.7.0-r0)
(3/6) Installing libcurl (7.51.0-r0)
(4/6) Installing expat (2.1.1-r2)
(5/6) Installing pcre (8.38-r1)
(6/6) Installing git (2.8.3-r0)
Executing busybox-1.24.2-r12.trigger
Executing ca-certificates-20160104-r4.trigger
OK: 22 MiB in 17 packages
 ---> b4b0f356c8ac
Removing intermediate container 87b47f8d5d9b
Step 3 : ADD static /static
 ---> 311fc8862edb
Removing intermediate container aba0b170b13a
Step 4 : ADD controller /bin/controller
 ---> cd104f6babef
Removing intermediate container dfbaf72eaa7d
Step 5 : EXPOSE 8080
 ---> Running in da52ef41d74f
 ---> dbab862fb18b
Removing intermediate container da52ef41d74f
Step 6 : ENTRYPOINT /bin/controller
 ---> Running in b2e3af6bec3d
 ---> 86937d8db64a
Removing intermediate container b2e3af6bec3d
Successfully built 86937d8db64a
{% endhighlight %}


{% highlight shell %}
docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED              STATUS              PORTS                                            NAMES
4be692396011        shipyard/shipyard:v1           "/bin/controller serv"   4 seconds ago        Up 2 seconds        0.0.0.0:8080->8080/tcp                           shipyard-controller
a3fdabe80a3f        swarm:latest                   "/swarm join --addr 1"   39 seconds ago       Up 38 seconds       2375/tcp                                         shipyard-swarm-agent
f42ec4a5c3d7        swarm:latest                   "/swarm manage --host"   42 seconds ago       Up 40 seconds       2375/tcp                                         shipyard-swarm-manager
89743f1b4db8        shipyard/docker-proxy:latest   "/usr/local/bin/run"     About a minute ago   Up About a minute   0.0.0.0:2375->2375/tcp                           shipyard-proxy
ee353d533403        microbox/etcd                  "/bin/etcd -name disc"   3 minutes ago        Up 3 minutes        0.0.0.0:4001->4001/tcp, 0.0.0.0:7001->7001/tcp   shipyard-discovery
66e947f0cf31        rethinkdb                      "rethinkdb --bind all"   4 minutes ago        Up 4 minutes        8080/tcp, 28015/tcp, 29015/tcp                   shipyard-rethinkdb
{% endhighlight %}

等待程序启动

{% highlight shell %}
[root@node0 controller]# docker logs 4be692396011
INFO[0000] shipyard version 3.1.0                       
INFO[0034] checking database                            
INFO[0357] created admin user: username: admin password: shipyard 
INFO[0357] controller listening on :8080          
{% endhighlight %}


http://192.168.1.130:8080/

![1]({{ site.url }}/pic/docker-shipyard/QQ截图20170105170016.png)

![1]({{ site.url }}/pic/docker-shipyard/123.png)

-----------------

end

