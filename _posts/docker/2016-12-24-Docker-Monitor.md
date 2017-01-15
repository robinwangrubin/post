---
layout: post
title: "Docker 1.12 Swarm集群实战 第六章"
categories: Docker
---


* content
{:toc}

## 前言

这一章我们来介绍如何使用cAdvisor+InfluxDB+Grafana搭建一个简单的swarm性能监控平台；

* cAdvisor：用来收集docker容器内部和host主机上的性能数据；
* InfuxDB：开源分布式时序数据库, 用来保存cAvisor收集的性能数据；
* Grafana：性能绘图仪表盘工具, 读取Influxdb性能数据,绘图展示；

## InfluxDB

启动一个lnfuxDB容器，用于收集docker性能数据

发布两个端口：8086用于lnfluxdb数据读写；8083用于数据库管理；

{% highlight shell %}
docker service create --network logging -p 8083:8083 -p 8086:8086 --mount source=influxdb-vol,type=volume,target=/var/lib/influxdb --name=influxdb --constraint 'node.hostname==node3' influxdb:alpine
{% endhighlight %}

这里使用logging网络, 如果你的swarm集群里面没有创建这个网络,使用命令创建：

> docker network create –driver overlay logging

influxdb启动成功后，访问集群任意节点ip的8083端口即可访问管理界面；

点击右上角的“连接设置”，输入用户名root，输入密码root，save保存

![1]({{ site.url }}/pic/docker-monitor/1.png)


下面我们创建一个数据库cadvisor, 用于保存swarm集群性能数据

在Query框中输入, CREATE DATABASE “cadvisor”按回车执行

![1]({{ site.url }}/pic/docker-monitor/b368eca9-1c0e-45d6-9532-dd28e69f7231.png)


SHOW DATABASES将会看到我们新建的数据库

![1]({{ site.url }}/pic/docker-monitor/141584c6-d026-485d-9e65-8adfdc4f0bbb.png)


------------------------

## cAdvisor

Google开源的容器监控工具；在Goolge开源社区关注度排名第一的工具；

需在所有swarm节点运行cAdvisor服务

1.任意一台节点下载cadvisior镜像

{% highlight shell %}
docker pull google/cadvisor
{% endhighlight %}

2.retag完上传至本地仓库便于其他节点快速下载

{% highlight shell %}
docker push localhost:5000/cadvisor
{% endhighlight %}

3.使用global模式在所有节点上部署cAdvisor

{% highlight shell %}
docker service create --network logging --name cadvisor --mode global \
--mount source=/var/run,type=bind,target=/var/run,readonly=false \
--mount source=/,type=bind,target=/rootfs,readonly=true \
--mount source=/sys,type=bind,target=/sys,readonly=true \
--mount source=/var/lib/docker,type=bind,target=/var/lib/docker,readonly=true \
localhost:5000/cadvisor -storage_driver=influxdb -storage_driver_host=influxdb:8086 \
-storage_driver_db=cadvisor
{% endhighlight %}

> \-\-mode global 指定service运行在每个swarm节点上

> \-\-mount 挂载本地docker socket用于监控docker性能

> -storage_driver=influxdb 指定存储驱动,使cadvisor将数据存储到数据库中

> -storage_driver_host=influxdb:8086 InfluxDB地址

> -storage_driver_db=cadvisor 数据库名称

4.验证启动情况

{% highlight shell %}
docker service ps cadvisor
ID                         NAME          IMAGE                    NODE   DESIRED STATE  CURRENT STATE          ERROR
cnenwrggfqu2dqaiwu9z132l0  cadvisor      localhost:5000/cadvisor  node2  Running        Running 3 minutes ago  
1eiyhx4xgz0dvsle9dlufpeyw   \_ cadvisor  localhost:5000/cadvisor  node1  Running        Running 3 minutes ago  
1xo08qgm9rwn0hjwsfxr9w7tp   \_ cadvisor  localhost:5000/cadvisor  node0  Running        Running 3 minutes ago  
ai2bmutq4g2nzinouhvkpvuug   \_ cadvisor  localhost:5000/cadvisor  node3  Running        Running 3 minutes ago  
avxzu8ab7mtyig4dbo18oxdss   \_ cadvisor  localhost:5000/cadvisor  node4  Running        Running 3 minutes ago  
{% endhighlight %}

### 验证cadvisor

打开InfluxDB的管理界面, 查询cadvisor数据库数据, 验证性能数据收集情况.

点击右上角切换数据库至cadvisor, 输入查询SHOW MEASUREMENTS.

![1]({{ site.url }}/pic/docker-monitor/826491f7-de37-4f69-969a-ea5fd5f4d026.png)


如果想查看具体的数据值可以使用select * from 查询

例如：例如我们查询下 memory_usage. 可以直接输入select * from memory_usage

----------------

## Grafana

创建Grafana服务

{% highlight shell %}
docker service create --network logging \
-p 3000:3000 -e "GF_SECURITY_ADMIN_PASSWORD=admin" \
--constraint 'node.hostname==node1' \
--name grafana grafana/grafana
{% endhighlight %}

打开浏览器访问swarm集群的3000端口,打开grafana webUI. 输入用户名密码登录

![1]({{ site.url }}/pic/docker-monitor/85cceb0a-e6b5-40b8-9579-43ec774a5829.jpg)


### 添加数据源

点击左上角图标, 选择Data Sources, 然后点击Add data source, 点击Default

![1]({{ site.url }}/pic/docker-monitor/9c53b468-ae1d-4fef-bf17-edb3ca04603c.png)

![1]({{ site.url }}/pic/docker-monitor/24de34ef-5975-452f-adc4-8dfb0d8a53b7.png)


### 新建性能绘图

以node1为例, 我们新建一个dashboard名为node1.

> Filesystem Limit/Usage

点击Home, 点击Create New, 点击右侧绿色块选Add Panel, 选Graph

![1]({{ site.url }}/pic/docker-monitor/62b772c9-fe6e-408c-b981-33be49063cf5.png)


General面板 Title填Filesystem Limit/Usage, Span填6

Metrics面板 Panel data source选Influxdb_source, 点击Add query,在上面的A, B查询框中分别输入查询:


{% highlight shell %}
SELECT mean("value") FROM "fs_limit" WHERE "com.docker.swarm.node.id" = '8ko8h0egr0vgydoww6poysg70' AND $timeFilter GROUP BY time($interval) fill(null)
SELECT mean("value") FROM "fs_usage" WHERE "com.docker.swarm.node.id" = '8ko8h0egr0vgydoww6poysg70' AND $timeFilter GROUP BY time($interval) fill(null)
{% endhighlight %}

Note: 8ko8h0egr0vgydoww6poysg70 是我的node1的ID，请根据实际情况修改（docker node ls 查看nodeID）

![1]({{ site.url }}/pic/docker-monitor/69604c2b-bfcd-4204-a40a-79605c38403e.png)


Axes面板, Left Y->Unit->data->bytes, Right Y->Unit->data->bytes 保存

![1]({{ site.url }}/pic/docker-monitor/035f25b4-535c-4de6-ac3f-73aa4af20886.png)

回到dashboard可以看到我们的监控绘图：

![1]({{ site.url }}/pic/docker-monitor/7fd47ea5-c416-409f-a732-525b0e30a444.png)

> CPU Usage

点击左上角绿色隐藏模块；选Add Panel, 选Graph

General面板 Title填CPU Usage, Span填6

Metrics面板 Panel data source选Influxdb_source, 查询窗口输入如下语句:

{% highlight shell %}
SELECT mean("value") FROM "cpu_usage_system" WHERE "container_name" != '/' AND "com.docker.swarm.node.id" = '8ko8h0egr0vgydoww6poysg70' AND $timeFilter GROUP BY time($interval), "container_name" fill(null)
{% endhighlight %}

Axes面板, Left Y->Unit->time->Hertz(1/s), Right Y->Unit->time->Hertz(1/s) 保存

> Memory Usage

点击右侧Add rows, 点击左侧绿色块选Add Panel, 选Graph

General面板 Title填Memory Usage, Span填6

Metrics面板 Panel data source选Influxdb_source, 查询窗口输入如下语句

{% highlight shell %}
SELECT mean("value") FROM "memory_usage" WHERE "com.docker.swarm.node.id" = '8ko8h0egr0vgydoww6poysg70' AND $timeFilter GROUP BY time($interval), "container_name" fill(null)
{% endhighlight %}

Axes面板, Left Y->Unit->time->bytes, Right Y->Unit->time->bytes 保存

> Network Transmit / Receive

点击左侧绿色块选Add Panel, 选Graph

General面板 Title填Network Transmit / Receive, Span填6

Metrics面板 Panel data source选Influxdb_source, 点击Add query,在上面的A, B查询框中分别输入查询:

{% highlight shell %}
SELECT mean("value") FROM "rx_bytes" WHERE "com.docker.swarm.node.id" = '8ko8h0egr0vgydoww6poysg70' AND $timeFilter GROUP BY time($interval) fill(null)
SELECT mean("value") FROM "tx_bytes" WHERE "com.docker.swarm.node.id" = '8ko8h0egr0vgydoww6poysg70' AND $timeFilter GROUP BY time($interval) fill(null)
{% endhighlight %}

Axes面板, Left Y->Unit->data rate->bytes/sec, Right Y->Unit->time->bytes/sec 保存

最后完成我们就可以看到这个节点的性能数据了：

![1]({{ site.url }}/pic/docker-monitor/5a308da1-7801-42f4-993e-f2dc1246a65e.png)


> 其他节点监控

你可以点击左上齿轮图标选Save As保存, 复制一份node1的配置. 然后编辑每个图表的查询语句替换com.docker.swarm.node.id, 就可以了.

-----------------------

Grafana 的绘图功能很强大, 而且支持多重数据源, 本章只是简单介绍下几个图的绘制方法,可能并没有什么实际意义, 只是希望能给大家一个入门的了解.

