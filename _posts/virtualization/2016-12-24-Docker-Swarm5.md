---
layout: post
title: "Docker Swarm 5 -- 补充说明"
categories: Docker
---


* content
{:toc}


## 前言

这一章节主要是对前面章节的补充说明；主要包括constraints约束和volume的挂载使用；

## constraints约束

还记得在前面我们创建的registry服务的时候,有一个问题点需要注意：

我们的registry是由swarm自动调度到某个节点上的. 这样的话我们如果我们重启service以后, registry服务可能会被启动再随机的节点.

造成我们上传的镜像都不见了. 如何解决这个问题呢?

在创建service的时候可以使用–constraints参数,后面跟表达式,限制service容器在每个节点的调度情况.比如你想指定service运行在某个节点上等. 例如指定service运行在node1上:

{% highlight shell %}
docker service create --name registry --publish 5000:5000 \
--constraint 'node.hostname==node1' registry
{% endhighlight %}

除了hostname也可以使用其他节点属性来创建约束表达式；写法参见下面：

* node.id

> node.id == 2ivku8v2gvtg4

* node.hostname

> node.hostname != node02

* node.role

> node.role == manager

* node.labels

> node.labels.security == high

* engine.labels

> engine.labels.operatingsystem == ubuntu 14.04

用户自定义labels可以使用docker node update命令添加, 例如:

{% highlight shell %}
docker node update --label-add security=high node3
{% endhighlight %}

查看自定义labels

{% highlight shell %}
docker node inspect node3
[
    {
        "ID": "jfasdjfirfjfijeia03012fa0",
...
        "Spec": {
            "Labels": {
                "security": "high"
            },
            "Role": "manager",
            "Availability": "active"
        },
        "Description": {
            "Hostname": "node3",
            "Platform": {
                "Architecture": "x86_64",
                "OS": "linux"
            },
...
    }
]
{% endhighlight %}

对于已有service, 可以通过docker service update,添加constraint配置, 例如:

{% highlight shell %}
docker service update registry \
--constraint-add 'node.labels.security==high'
{% endhighlight %}

--------------------------

## volume创建管理

有了service约束, 我们可以保证我们的registry服务, 一直在node01节点上了.

不过还有一个问题, 就是如果我们删除了registry服务. 那我们上传的容器镜像也就被删除了.

如何保证即使registry服务被删除, 镜像可以保留呢?

这里我们可以使用 docker volume指定挂载一个数据卷用来保存镜像, 即使registry服务被删除了. 我们重新启动一个服务, 挂载这个数据卷. 我们上传的镜像还可以保存的.

在swarm集群中我们可以创建本地卷或者全局卷来挂载到容器, 用来保存数据.


* 全局卷可以被挂载在swarm集群的任意节点, 所以不管你的服务容器启动在哪个节点, 都可以访问到数据. 不过docker目前还没有默认的全局卷驱动支持, 你可以安装一些插件驱动来实现全局卷例如Flocker, Portworx等.

* 本地卷就只存在与某个节点本地的一个挂载卷.

为我们刚刚新建的registry服务, 挂载一个本地卷,可以使用如下命令:

{% highlight shell %}
docker service update registry \
--mount-add type=volume,source=registry-vol,target=/var/lib/registry
{% endhighlight %}

source=registry-vol 中registry-vol为卷名字, 执行上述命令以后,docker会自动为我们创建一个registry-vol本地卷.

可以使用docker volume ls命令查看:

{% highlight shell %}
docker volume ls
DRIVER              VOLUME NAME
local               registry-vol

docker volume inspect registry-vol
[
    {
        "Name": "registry-vol",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/registry-vol/_data",
        "Labels": null,
        "Scope": "local"
    }
]
{% endhighlight %}

上面命令, 可以看到本机卷挂载到节点的目录.

这样即使我们现在删除registry服务. 也可以只用如下命令重新创建一个registry服务, 挂载registry-vol来找回我们的镜像.

{% highlight shell %}
docker service create --name registry --publish 5000:5000 \
--mount source=registry-vol,type=volume,target=/var/lib/registry \
-e SEARCH_BACKEND=sqlalchemy \
--constraint 'node.hostname==node3' registry
{% endhighlight %}

-------------------

end

