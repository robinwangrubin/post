---
layout: post
title: Prometheus-容器监控
categories: Docker
---


* content
{:toc}


> 首先-什么是 TSDB (Time Series Database):

我们可以简单的理解为.一个优化后用来处理时间序列数据的软件,并且数据中的数组是由时间进行索引的.

> 时间序列数据库的特点:

* 大部分时间都是写入操作
* 写入操作几乎是顺序添加;大多数时候数据到达后都以时间排序.
* 写操作很少写入很久之前的数据,也很少更新数据.大多数情况在数据被采集到数秒或者数分钟后就会被写入数据库.
* 删除操作一般为区块删除,选定开始的历史时间并指定后续的区块.很少单独删除某个时间或者分开的随机时间的数据.
* 数据一般远远超过内存大小,所以缓存基本无用.系统一般是 IO 密集型
* 读操作是十分典型的升序或者降序的顺序读,
* 高并发的读操作十分常见.


> 常见的时间序列数据库:

* influxDB	https://influxdata.com/
* RRDtool	http://oss.oetiker.ch/rrdtool/
* Graphite	http://graphite.readthedocs.org/en/latest/
* OpenTSDB	http://opentsdb.net/
* Kdb+	http://kx.com/
* Druid	http://druid.io/
* KairosDB	http://kairosdb.github.io/
* Prometheus	https://prometheus.io/



## 什么是Prometheus

Prometheus是由 SoundCloud 开发的开源监控报警系统和时序列数据库(TSDB)，可以看作是Google内部监控系统Borgmon的一个（非官方）实现。

并且保持独立于任何公司,Prometheus 在2016加入 CNCF ( Cloud Native Computing Foundation ), 作为在 kubernetes 之后的第二个由基金会主持的项目.

为什么选择Prometheus而不是其它TSDB实现（如InfluxDB）?

主要是因为Prometheus的核心功能，查询语言PromQL，它更像一种可编程计算器，而不是其那么像SQL，也意味着PromQL可以近乎无限之组合出各种查询结果

## 主要特点

1. 多维数据模型（时序列数据由metric名和一组key/value组成）
2. 在多维度上灵活的查询语言(PromQl)
3. 不依赖分布式存储，单主节点工作.
4. 通过基于HTTP的pull方式采集时序数据
5. 可以通过中间网关进行时序列数据推送(pushing)
6. 目标服务器可以通过发现服务或者静态配置实现
7. 多种可视化和仪表盘支持


## 相关组件

Prometheus生态系统由多个组件组成，其中许多是可选的：

* Prometheus server

主要负责数据采集和存储，提供PromQL查询语言的支持

* 客户端sdk

官方提供的客户端类库有go、java、scala、python、ruby，其他还有很多第三方开发的类库，支持nodejs、php、erlang等


* Push Gateway

支持临时性Job主动推送指标的中间网关

* PromDash

使用rails开发的dashboard，用于可视化指标数据；目前普遍用Grafana代替

* exporters

支持其他数据源的指标导入到Prometheus，支持数据库、硬件、消息中间件、存储系统、http服务器、jmx等


* alertmanager

实验性的报警管理端(alartmanager,单独进行报警汇总,分发,屏蔽等 )


promethues 的各个组件基本都是用 golang 编写,对编译和部署十分友好.并且没有特殊依赖.基本都是独立工作.


## prometheus架构

-----------------------

![1]({{ site.url }}/pic/prometheus/1.svg)

Prometheus的基本原理是通过HTTP协议周期性抓取被监控组件的状态，任意组件只要提供对应的HTTP接口就可以接入监控。不需要任何SDK或者其他的集成过程。这样做非常适合做虚拟化环境监控系统，

比如VM、Docker、Kubernetes等。输出被监控组件信息的HTTP接口被叫做exporter 。目前互联网公司常用的组件大部分都有exporter可以直接使用，比如Varnish、Haproxy、Nginx、MySQL、Linux系统信息(包括磁盘、内存、CPU、网络等等)。

> Prometheus服务过程大概是这样：

1. Prometheus Daemon负责定时去目标上抓取metrics(指标)数据，每个抓取目标需要暴露一个http服务的接口给它定时抓取。Prometheus支持通过配置文件、文本文件、Zookeeper、Consul、DNS SRV Lookup等方式指定抓取目标。Prometheus采用PULL的方式进行监控，即服务器可以直接通过目标PULL数据或者间接地通过中间网关来Push数据。

2. Prometheus在本地存储抓取的所有数据，并通过一定规则进行清理和整理数据，并把得到的结果存储到新的时间序列中。

3. Prometheus通过PromQL和其他API可视化地展示收集的数据。Prometheus支持很多方式的图表可视化，例如Grafana、自带的Promdash以及自身提供的模版引擎等等。Prometheus还提供HTTP API的查询方式，自定义所需要的输出。

4. PushGateway支持Client主动推送metrics到PushGateway，而Prometheus只是定时去Gateway上抓取数据。

5. Alertmanager是独立于Prometheus的一个组件，可以支持Prometheus的查询语句，提供十分灵活的报警方式。


## prometheus部署

{% highlight shell %}
docker run \
	-p 9090:9090 \
	-v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml \
	-v /home/mtime/prometheus-docker/data：/prometheus/data \
	prom/prometheus
{% endhighlight %}



