---
layout: post
title: "Docker 1.12 Swarm集群实战 第五章"
categories: Docker
---


* content
{:toc}

## 前言

现在我们有了docker swarm集群，上面跑了docker币应用，以后还会有更多的应用跑在集群上；

如何监控每个应用的运行状态呢？

所以我们需要一个日志平台用来收集分析展示所有的swarm集群上应用的日志;

## ELK日志平台

在本小节我们使用ELK日志平台，什么是ELK？

* ElasticSearch：存储和索引日志
* Logstash：接收，发送，过滤，分隔日志
* Kibana：用来索引，展示，分析日志的UI

我们会在每个swarm节点上使用syslgo协议发送日志到logstash，存储在elasticsearch中；最后使用kibana分析展示日志。

----------------------------

Note: 后续补充


