---
layout: post
title: "Docker Swarm基础"
categories: Docker
---

* content
{:toc}

## Reference

> https://docs.docker.com/engine/swarm/

------------------

## Swarm mode overview

Docker 1.12 内置了集群管理工具swarm，可使用docker的CLI去创建swarm集群；管理swarm集群；部署应用服务。

### Feature highlights

> 引擎内置集群管理工具：使用docker的CLI即可创建管理集群；部署应用；无需额外的编排软件管理集群。

> 分散式设计：不是在部署时处理节点角色之间的差异，而是在运行时进行专业化分工。 你可以使用Docker Engine部署两种类型的节点，manager和worker。 这意味着您可以从单个磁盘映像构建整个群集。

> 声明性服务模型：Docker Engine使用声明性方法来定义应用程序堆栈中各种服务的所需状态。 例如，你可以描述由具有消息队列服务和数据库后端的Web前端服务组成的应用程序。

> Scaling：你可以声明每个服务运行的副本数量，当你想要扩容或收缩时，swarm管理器通过自动添加或删除任务来适应所需的目标状态。

> 期望状态协调：manager节点实时监视群集状态，并协调你声明的期望状态的实际状态之间的任何差异。 例如，如果你设置一个服务运行10个副本，如果其中两个副本所在计算机崩溃，manager则自动在可用的worker节点创建两个新的任务，以达到你希望的目标状态（10个副本）。

> 多主机网络：你可以为你的服务创建overlay网络，manager自动分配地址给容器在初始化或更新应用容器的时候。

> 服务发现：manager自动为每个服务分配一个唯一的DNS名称，并对运行的容器负载均衡，你可以通过嵌入在swarm中的DNS服务器查询每个容器的域名信息。

> 负载均衡：你可以暴露一个服务端口给外部的负载均衡器，在swarm内部。你可以指定如何在节点之间分发任务。
安全：swarm节点之间之间采用TLS认证通信，你可以选择使用自签名或CA颁发的证书

> 滚动更新：你可以滚动更新你的应用，手动控制更新时的延时时间，如果出现问题，你可以回滚应用到之前的版本。

--------------------------

## Swarm mode key concepts

### Swarm

When you run Docker Engine outside of swarm mode, you execute container commands. When you run the Engine in swarm mode, you orchestrate services.

### Node

一个node就是一个docker实例；

当你部署服务时，你在manager node定义服务（一个服务可能由多个task共同组成），manager节点将tasks调度到worker节点执行。

manager节点还负责维护集群管理功能使集群达到期望状态，manager节点中间会挑选一个leader出来负责编排任务。

woker节点负责接收并执行manager分配的任务，默认情况下，manager节点也是worker节点，但是你可以设置让manager节点仅仅作为manager

### Services and tasks

service就是被定义在worker上执行的task；

当你创建service时，你指定所需的image和需在容器中执行的command

在replicated services 模式，manager会根据你设置的副本数量自动分配task

task就是一个容器，也是swarm调度的原子单位，manager节点根据你设置的副本数量分配任务到worker节点。当一个任务分配到node上，他就只能是runing或fail的状态。

### Load balancing

manager使用ingress load balancing对外暴露服务，manager会自动分配PublishedPort 或者你也可以手动指定未被使用的端口号，如果是你不手动指定，swarm会自动在30000-32767中间分配一个端口号

你可以访问任意节点的PublishedPort，即使该节点上并没有任务相应的task在运行

Swarm有一个内部的DNS组件会自动为每个任务分配一条DNS记录，manager根据DNS域名负载该服务的所有请求。

------------------

## Getting started with swarm mode

### environment

host：

* node0 manager1 192.168.1.130
* node1	manager2 192.168.1.131
* node2	manager3 192.168.1.132
* node4	worker1	192.168.1.133
* node5	worker2	192.168.1.134

docker-version：

{% highlight shell %}
Client:
 Version:      1.12.3
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   6b644ec
 Built:        
 OS/Arch:      linux/amd64

Server:
 Version:      1.12.3
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   6b644ec
 Built:        
 OS/Arch:      linux/amd64
{% endhighlight %}

port：

* TCP port 2377 for cluster management communications
* TCP and UDP port 7946 for communication among nodes
* TCP and UDP port 4789 for overlay network traffic

-------------------------

## Create a swarm

> docker swarm init –advertise-addr

{% highlight shell %}
[root@node0 ~]# docker swarm init --advertise-addr 192.168.1.130
Swarm initialized: current node (36duyub88vg54x41e0dur9850) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1n11n79rldx65uq8krc7h41irve0j4x6lmrlea33u2wriw8vym-8wfukcb9amac6wkcgm340aj6f \
    192.168.1.130:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
{% endhighlight %}

> docker info

{% highlight shell %}
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 35
Server Version: 1.12.3
...snip...
Swarm: active
 NodeID: 36duyub88vg54x41e0dur9850
 Is Manager: true
 ClusterID: 4uxe2ejgkqky5x77g57p4cp3s
 Managers: 1
 Nodes: 1
 ...snip...
{% endhighlight %}

> docker node ls

{% highlight shell %}
[root@node0 ~]# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
36duyub88vg54x41e0dur9850 *  node0     Ready   Active        Leader
{% endhighlight %}

* 表示你当前连接的主机；swarm自动用主机名命名node

-------------------

## Join nodes to a swarm

docker加入到集群是依赖`join-token`的，node使用不同的token可以以不同的身份加入到集群中；如果你后来变更了token，也不影响已经加入到集群的node。

### Join as a worker node

> docker swarm join-token worker

{% highlight shell %}
[root@node0 ~]# docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1n11n79rldx65uq8krc7h41irve0j4x6lmrlea33u2wriw8vym-8wfukcb9amac6wkcgm340aj6f \
    192.168.1.130:2377
{% endhighlight %}

### Join as a manager node

> docker swarm join-token manager

{% highlight shell %}
[root@node0 ~]# docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-1n11n79rldx65uq8krc7h41irve0j4x6lmrlea33u2wriw8vym-bdik5uby0lq4s955ij8gnrtkb \
    192.168.1.130:2377
{% endhighlight %}

----------------

* switches the Docker Engine on the current node into swarm mode.
* requests a TLS certificate from the manager.
* names the node with the machine hostname
* joins the current node to the swarm at the manager listen address based upon the swarm token.
* sets the current node to Active availability, meaning it can receive tasks from the scheduler.
* extends the ingress overlay network to the current node.

---------------

## Manage nodes in a swarm

### List nodes

{% highlight shell %}
[root@node0 ~]# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
36duyub88vg54x41e0dur9850 *  node0     Ready   Active        Leader
4bgh5mz90cc2cni8vn7ur9gb4    node2     Ready   Active        
9sh9731bp74enjx9nht6irbon    node3     Ready   Active        
czavy0nt0bro4fj1tl15fby62    node4     Ready   Active        
dazpqp2i92e6ityg064r75ahq    node1     Ready   Active    
{% endhighlight %}

AVAILABILITY:

* Active 可以分配任务到该节点
* Pause 不会分配新的任务到该节点，已有的任务继续运行
* Drain 不会分配新的任务到该节点，已有的任务调度到其他node上运行

MANAGER STATUS：

* 没有值表示是worker节点，不参与选举。
* Leader 表示是主管理节点
* Reachable 表示是管理节点，当Leader挂掉后会参与选举。
* Unavailable 表示该node是管理节点，但是与其他的manager失去了联系。这时候应该将其降权，并提升一名worker作为新manager

### Inspect an individual node

> docker node inspect --pretty

{% highlight shell %}
docker node inspect node0 --pretty
ID:                     36duyub88vg54x41e0dur9850
Hostname:               node0
Joined at:              2017-01-01 07:18:11.827960041 +0000 utc
Status:
 State:                 Ready
 Availability:          Active
Manager Status:
 Address:               192.168.1.130:2377
 Raft Status:           Reachable
 Leader:                Yes
Platform:
 Operating System:      linux
 Architecture:          x86_64
Resources:
 CPUs:                  1
 Memory:                977.9 MiB
Plugins:
  Network:              bridge, host, null, overlay
  Volume:               local
Engine Version:         1.12.3
{% endhighlight %}

### Change node availability

> docker node update

{% highlight shell %}
docker node update --availability drain node0
node0
{% endhighlight %}


### Add or remove label metadata


{% highlight shell %}
docker node update --label-add foo --label-add server=java node0
node0
{% endhighlight %}


### Promote or demote a node

> docker node promote

{% highlight shell %}
# docker node promote node1 node2
Node node1 promoted to a manager in the swarm.
Node node2 promoted to a manager in the swarm.
# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
36duyub88vg54x41e0dur9850 *  node0     Ready   Drain         Leader
4bgh5mz90cc2cni8vn7ur9gb4    node2     Ready   Active        Reachable
9sh9731bp74enjx9nht6irbon    node3     Ready   Active        
czavy0nt0bro4fj1tl15fby62    node4     Ready   Active        
dazpqp2i92e6ityg064r75ahq    node1     Ready   Active        Reachable
{% endhighlight %}

> docker node demote


{% highlight shell %}
# docker node demote node1 node2
Manager node1 demoted in the swarm.
Manager node2 demoted in the swarm.
# docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
36duyub88vg54x41e0dur9850 *  node0     Ready   Drain         Leader
4bgh5mz90cc2cni8vn7ur9gb4    node2     Ready   Active        
9sh9731bp74enjx9nht6irbon    node3     Ready   Active        
czavy0nt0bro4fj1tl15fby62    node4     Ready   Active        
dazpqp2i92e6ityg064r75ahq    node1     Ready   Active  
{% endhighlight %}

### Leave the swarm

> docker swarm leave


worker离开集群

{% highlight shell %}
docker swarm leave
Node left the swarm.
{% endhighlight %}

manager离开集群

先降权成为worker，在以worker身份离开集群

当node离开了集群之后，可以删除node节点

> docker node rm

{% highlight shell %}
docker node rm node1
{% endhighlight %}

--------------

## Deploy a service to the swarm

> docker service create

{% highlight shell %}
docker service create --replicas 1 --name helloworld alpine ping docker.com
{% endhighlight %}

> docker service ls

--------------------

## Inspect a service on the swarm

> docker service inspect –pretty

{% highlight shell %}
docker service inspect --pretty helloworld
{% endhighlight %}

> docker service ps

{% highlight shell %}
docker service ps helloworld
{% endhighlight %}

-----------------------------

## Scale the service in the swarm

> docker service scale

{% highlight shell %}
docker service scale helloworld=5
{% endhighlight %}

> docker service ps

{% highlight shell %}
docker service ps helloworld
{% endhighlight %}

---------------

## Delete the service running on the swarm

> docker service rm

{% highlight shell %}
docker service rm helloworld
{% endhighlight %}

----------------

## Apply rolling updates to a service

部署 Redis 3.0.6 并设置update服务时的延时为10s

{% highlight shell %}
docker service create \
  --replicas 3 \
  --name redis \
  --update-delay 10s \
  redis:3.0.6
{% endhighlight %}

`--update-delay`标签用来配置update服务时的延时间隔；可选格式：3h 5m 10m30s 10s

默认情况下一次升级所有服务，你可以使用`--update-parallelism`标签设置一次更新任务的数量

默认情况下，更新任务中如果有失败的情况，会停止继续更新，当然，你可以通过`--update-failure-action`标签去控制更新失败后的行为

Inspect the redis service:

{% highlight shell %}
docker service inspect --pretty redis
{% endhighlight %}

updates service：

{% highlight shell %}
docker service update –image redis:3.0.7 redis
{% endhighlight %}

调度更新过程如下：

停止第一个任务

升级刚才停止的任务

启动新的容器替换刚才停止任务

如果更新状态是RUNNING，等待延时间隔，续集更新下一个任务

如果跟新状态是FAILED，停止更新

Run docker service inspect –pretty redis to see the new image in the desired state:

{% highlight shell %}
docker service inspect --pretty redis
{% endhighlight %}

To restart a paused update run docker service update . For example:

{% highlight shell %}
docker service update redis
{% endhighlight %}

Run docker service ps to watch the rolling update:

{% highlight shell %}
docker service ps redis
{% endhighlight %}

-------------------------

## Drain a node on the swarm

默认情况下，swarm manager会向所有状态为`ACTIVE` 的节点分配任务，包括manager自身。

当你需要使某台设备下架维护时，你可以将这台设备设置为`DRAIN`状态，那么这台设备上的任务会被停止掉，自动在状态为 `ACTIVE`的节点上启动该任务。

> docker node update –availability drain

> docker node update –availability active

---------------------------

## Use swarm mode routing mesh

swarm使对外暴露服务端口变得简单，所有的节点加入到了ngress routing mesh，routing mesh可以使每个节点都能接收外部的请求，即使某个节点上没有相关任务在运行，routing mesh也能正确的将请求路由到可用的节点。

为了使用 ingress network 你需要占用以下端口:

* Port 7946 TCP/UDP for container network discovery.
* Port 4789 UDP for the container ingress network.

### Publish a port for a service

{% highlight shell %}
docker service create \
  --name <SERVICE-NAME> \
  --publish <PUBLISHED-PORT>:<TARGET-PORT> \
  <IMAGE>
{% endhighlight %}

`<TARGET-PORT>`表示容器监听的端口，`<PUBLISHED-PORT>` swarm服务监听的端口

例如：想对外提供8080的访问

{% highlight shell %}
docker service create \
  --name my-web \
  --publish 8080:80 \
  --replicas 2 \
  nginx
{% endhighlight %}

当你访问集群任意节点的8080端口，swarm自动负载路由请求到可用节点上。

![1]({{ site.url }}/pic/Docker-Swarm-Basic/ingress-routing-mesh.png)


对于已经存在的服务，你可以使用update去更新对外端口

{% highlight shell %}
docker service update \
  --publish-add <PUBLISHED-PORT>:<TARGET-PORT> \
  <SERVICE>
{% endhighlight %}

查看服务对外暴露的端口

{% highlight shell %}
docker service inspect --format="" my-web
{% endhighlight %}

暴露TCP端口

> docker service create –name dns-cache -p 53:53 dns-cache

> docker service create –name dns-cache -p 53:53/tcp dns-cache

暴露UDP端口

> docker service create –name dns-cache -p 53:53/udp dns-cache

暴露TCP/UDP端口

> docker service create –name dns-cache -p 53:53/tcp -p 53:53/udp dns-cache

### Configure an external load balancer

你可以配置一个外部的负载均衡器去路由请求到swarm节点，例如 HAProxy负载请求到nginx服务的8080端口

![1]({{ site.url }}/pic/Docker-Swarm-Basic/ingress-lb.png)


--------------------

end

