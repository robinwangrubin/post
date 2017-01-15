---
layout: post
title: "Docker 1.12 Swarm集群实战 第四章"
categories: Docker
---


* content
{:toc}


## 前言

通过上一章节的测试，我们怀疑是程序代码的问题导致瓶颈所在

这一章我们会修正代码，并学习如何利用docker swarm滚动更新我们的服务副本容器

### 应用源码分析

下面我们来一起看看rng和hasher应用的源代码

{% highlight shell %}
[root@node0 ~]# cd /root/orchestration-workshop/dockercoins/
[root@node0 dockercoins]# ls
docker-compose.yml  docker-compose.yml-images  docker-compose.yml-logging  hasher  rng  webui  worker
[root@node0 dockercoins]# cd rng/
[root@node0 rng]# cat rng.py 
from flask import Flask, Response
import os
import socket
import time
app = Flask(__name__)
# Enable debugging if the DEBUG environment variable is set and starts with Y
app.debug = os.environ.get("DEBUG", "").lower().startswith('y')
hostname = socket.gethostname()
urandom = os.open("/dev/urandom", os.O_RDONLY)
@app.route("/")
def index():
    return "RNG running on {}\n".format(hostname)
@app.route("/<int:how_many_bytes>")
def rng(how_many_bytes):
    # Simulate a little bit of delay
    time.sleep(0.1)
    return Response(
        os.read(urandom, how_many_bytes),
        content_type="application/octet-stream")
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
{% endhighlight %}

time.sleep(0.1)每个请求会自动延迟100ms响应.

hasher和worker的也一样

{% highlight shell %}
[root@node0 hasher]# cat hasher.rb 
require 'digest'
require 'sinatra'
require 'socket'
set :bind, '0.0.0.0'
set :port, 80
post '/' do
    # Simulate a bit of delay
    sleep 0.1
    content_type 'text/plain'
    "#{Digest::SHA2.new().update(request.body.read)}"
end
get '/' do
    "HASHER running on #{Socket.gethostname}\n"
end
{% endhighlight %}

现在你知道100ms延迟的原因了吧. 下面我们以worker服务为例, 更新下源代码缩短下响应时间.

## Rolling Updates

在swarm 集群的环境下, 我们每个服务都会有多个容器副本, 如何在不停止应用的情况下滚动更新每个容器副本就十分重要了

docker swarm集群为我们提供了方便的命令

那么如果我们要发布一个新版本的worker服务需要做什么呢:

1. 更新worker服务的源代码, 缩短time.sleep(0.1)到0.01
2. 重新build worker镜像, 使用一个新的tag版本
3. push新镜像到我们本地的镜像仓库
4. 滚动更新worker服务, 使用新的镜像

在开始之前, 如果你一直跟着文章做的, 上一章为了性能测试我们已经把worker服务副本scale成0了, 我们先恢复成10个副本.

{% highlight shell %}
[root@node0 ~]# docker service scale worker=10
[root@node0 ~]# docker service ls
ID            NAME      REPLICAS  IMAGE                                   COMMAND
0pcuhiw9jmu4  rng       5/5       localhost:5000/dockercoins_rng:v0.1     
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1  
3l07xdfj4rjk  redis     1/1       redis                                   
52c93icugean  webui     1/1       localhost:5000/dockercoins_webui:v0.1   
anm91nvl3sdm  debug     global    alpine                                  sleep 1000000000
cgk5fahusjsy  registry  1/1       registry                                
clvgacmx24p7  worker    10/10     localhost:5000/dockercoins_worker:v0.1
{% endhighlight %}

### 更新应用代码

下面我们来更新worker服务的代码~/orchestration-workshop/dockercoins/worker/worker.py 把sleep时间从0.1改成0.01

{% highlight shell %}
[root@node0 dockercoins]# cat worker/worker.py |grep time.sleep
    time.sleep(0.01)
    time.sleep(10)
{% endhighlight %}

### 重新build镜像

{% highlight shell %}
[root@node0 dockercoins]# docker build -t localhost:5000/dockercoins_worker:v0.01 worker
{% endhighlight %}

### push镜像到本地registry

{% highlight shell %}
[root@node0 dockercoins]# docker push localhost:5000/dockercoins_worker:v0.01
{% endhighlight %}

### 滚动更新worker服务

下面我们有了新worker服务的镜像v0.01, 我们来滚动更新我们的10个worker:

{% highlight shell %}
[root@node0 ~]# docker service update worker --update-parallelism 2 --update-delay 5s --image localhost:5000/dockercoins_worker:v0.01
{% endhighlight %}

\-\-update-parallelism：每次update的容器数量

\-\-update-delay：每次更新之后的等待时间

\-\-image：镜像名称

查看更新情况

{% highlight shell %}
[root@node0 ~]# docker service ls
ID            NAME      REPLICAS  IMAGE                                    COMMAND
0pcuhiw9jmu4  rng       5/5       localhost:5000/dockercoins_rng:v0.1      
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1   
3l07xdfj4rjk  redis     1/1       redis                                    
52c93icugean  webui     1/1       localhost:5000/dockercoins_webui:v0.1    
anm91nvl3sdm  debug     global    alpine                                   sleep 1000000000
cgk5fahusjsy  registry  1/1       registry                                 
clvgacmx24p7  worker    10/10     localhost:5000/dockercoins_worker:v0.01
{% endhighlight %}

![1]({{ site.url }}/pic/docker-swarm-4/308450367.png)


现在已经达到我们的预期值了：10个worker每秒产生40个docker币

## 容器服务回滚

如果我们发现, 新版本worker有问题希望回滚怎么办呢?

很简单, 跟上面更新的命令一样啊, 直接只用v0.1版本的镜像就可以了.

上面我们演示了滚动更新, 如果你希望一次性都更新或者回滚呢, 更简单了, 不加参数就行了.

下面我们一次性回滚所有worker服务到v0.1版本

{% highlight shell %}
[root@node0 dockercoins]# docker service update worker --image localhost:5000/dockercoins_worker:v0.1
worker
[root@node0 dockercoins]# docker service ls
ID            NAME      REPLICAS  IMAGE                                   COMMAND
0pcuhiw9jmu4  rng       5/5       localhost:5000/dockercoins_rng:v0.1     
22cli9o3u9ho  hasher    1/1       localhost:5000/dockercoins_hasher:v0.1  
3l07xdfj4rjk  redis     1/1       redis                                   
52c93icugean  webui     1/1       localhost:5000/dockercoins_webui:v0.1   
anm91nvl3sdm  debug     global    alpine                                  sleep 1000000000
cgk5fahusjsy  registry  1/1       registry                                
clvgacmx24p7  worker    10/10     localhost:5000/dockercoins_worker:v0.1  
{% endhighlight %}

可以看到worker服务都回滚到v0.1版本了

---------------

end

