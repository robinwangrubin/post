---
layout: post
title: "Docker -- 日志系统FKLEK"
categories: Docker
---


* content
{:toc}

## FKLEK系统介绍

* Fluentd: 专业的集中化容器日志收集工具
* Kafka：高吞吐异步消息队列
* Logstash：接收，发送，过滤，分隔日志
* ElasticSearch：存储和索引日志
* Kibana：用来索引，展示，分析日志的UI

1. Fluentd将所有容器的日志并转发到kafka上。
2. Kafka临时存储所有容器的日志；用于被Logstash消费
3. Logstash消费kafka的日志，最终转存到ElasticSearch上。
4. ElasticSearch永久存储日志的数据库
5. Kibana从ElasticSearch中抽取数据页面展示

Docker ==> Fluentd ==> Kafka ==> Logstash ==> ElasticSearch ==> Kibana

----------------------------

## 实验环境准备

* 系统要求：CentOS-7.2
* 内核要求：3.10.0-327.el7.x86_64
* Docker版本：17.03.1-ce
* JAVA环境：JDK_1.8
* Zookeeper版本：zookeeper-3.4.6
* kafka版本： 2.11-0.10.0.1
* elasticsearch版本：2.4
* logstash版本：2.4
* kibana版本：4.6
* fluentd版本：任意



IP地址和应用安装对应表

192.168.1.250 ===== kafka01 Fluentd(Container) Kibana(Container)

192.168.1.251 ===== kafka02 Fluentd(Container) ElasticSearch(Container)

192.168.1.252 ===== kafka03 Fluentd(Container) Logstash(Container)

------------------------------

## 系统环境准备

* 主机名解析

所有节点：
{% highlight shell %}
cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.250   kafka01
192.168.1.251   kafka02
192.168.1.252   kafka03
192.168.1.250 192.168.1.251 192.168.1.252 fluentd.test.com	#利用DNS轮询伪造fluentd集群
{% endhighlight %}


* JAVA_1.8环境部署

{% highlight shell %}
tar xf jdk-8u11-linux-x64.tar.gz -C /usr/local/
cat >> /etc/profile <<EOF
export JAVA_HOME=/usr/local/jdk1.8.0_11
export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar
export PATH=\$JAVA_HOME/bin:\$PATH
EOF
source /etc/profile
{% endhighlight %}

-------------------------------------------


kafka集群依赖于zookeeper，所以我们先搭建zookeeper集群

## 部署zookeeper集群

* 安装zookeeper

{% highlight shell %}
wget http://apache.fayea.com/zookeeper/zookeeper-3.4.6/zookeeper-3.4.6.tar.gz
tar xf zookeeper-3.4.6.tar.gz -C /usr/local/
ln -s /usr/local/zookeeper-3.4.6/ /usr/local/zookeeper
mv /usr/local/zookeeper/conf/zoo_sample.cfg /usr/local/zookeeper/conf/zoo.cfg
{% endhighlight %}

* 配置zookeeper

三台配置都如下：
{% highlight shell %}
cat /usr/local/zookeeper/conf/zoo.cfg 
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
clientPort=2181
server.1=192.168.1.250:3181:4181
server.2=192.168.1.251:3181:4181
server.3=192.168.1.252:3181:4181
{% endhighlight %}

{% highlight shell %}
mkdir -p /tmp/zookeeper/
echo 1 > /tmp/zookeeper/myid		//kafka01执行
echo 2 > /tmp/zookeeper/myid		//kafka02执行
echo 3 > /tmp/zookeeper/myid		//kafka03执行
{% endhighlight %}

* 启动zookeeper集群

> /usr/local/zookeeper/bin/zkServer.sh start /usr/local/zookeeper/conf/zoo.cfg

* 验证zookeeper集群状态

> /usr/local/zookeeper/bin/zkServer.sh status /usr/local/zookeeper/conf/zoo.cfg


--------------------------------------------

## 部署kafka集群

* 安装kafka

{% highlight shell %}
wget http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/0.10.0.1/kafka_2.11-0.10.0.1.tgz
tar xf kafka_2.11-0.10.0.1.tgz -C /usr/local/
ln -s /usr/local/kafka_2.11-0.10.0.1/ /usr/local/kafka
mkdir -p /var/log/kafka/
{% endhighlight %}

* 配置kafka

{% highlight shell %}
cat /usr/local/kafka/config/server.properties
broker.id=1			//每台kafka都要设置独一无二broker ID号
port=9092			//监听端口号；推荐三天机器都设置成一样的
host.name=192.168.1.250		//每台kafka设置成自己的IP地址
advertised.host.name=192.168.1.250		//每台kafka设置成自己的IP地址
zookeeper.connect=192.168.1.250:2181,192.168.1.251:2181,192.168.1.252:2181	//zookeeper地址；三台机器都设置成一样
log.dirs=/var/log/kafka
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
num.partitions=1
num.recovery.threads.per.data.dir=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connection.timeout.ms=6000
delete.topic.enable=true
{% endhighlight %}

* 启动kafka集群

{% highlight shell %}
nohup /usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties &>>/var/log/kafka/kafka-server.log  &
{% endhighlight %}


* 验证kafka集群

创建一个topic

{% highlight shell %}
/usr/local/kafka/bin/kafka-topics.sh -zookeeper 192.168.1.250:2181,192.168.1.251:2181,192.168.1.252:2181 -topic wangrubintest -replication-factor 2 -partitions 3 -create
{% endhighlight %}

查看topic列表

{% highlight shell %}
/usr/local/kafka/bin/kafka-topics.sh -zookeeper 192.168.1.250:2181,192.168.1.251:2181,192.168.1.252:2181 -list
{% endhighlight %}

创建测试生产者

{% highlight shell %}
/usr/local/kafka/bin/kafka-console-producer.sh -broker-list 192.168.1.250:9092,192.168.1.251:9092,192.168.1.252:9092 -topic wangrubintest
{% endhighlight %}

创建测试消费者

{% highlight shell %}
/usr/local/kafka/bin/kafka-console-consumer.sh --zookeeper 192.168.1.250:2181,192.168.1.251:2181,192.168.1.252:2181 --topic wangrubintest
{% endhighlight %}


------------------------------------------

## 部署Fluentd收集日志

* 编写Dockerfile文件

{% highlight shell %}
mkdir fluentd && cd fluentd
vim Dockerfile
FROM fluent/fluentd:onbuild
MAINTAINER YOUR_NAME <...@...>
USER root
RUN apk add --update --virtual .build-deps \
        sudo build-base ruby-dev \

 # cutomize following instruction as you wish
 && sudo -u fluent gem install \
        fluent-plugin-elasticsearch \
        fluent-plugin-kafka \

 && sudo -u fluent gem sources --clear-all \
 && apk del .build-deps \
 && rm -rf /var/cache/apk/* \
           /home/fluent/.gem/ruby/2.3.0/cache/*.gem
EXPOSE 24284
{% endhighlight %}

* 编写fluent配置文件

{% highlight shell %}
vim fluent.conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<match *.**>
  @type kafka_buffered		//该字段表示output给kafka

  # list of seed brokers
  brokers 192.168.1.250:9092,192.168.1.251:9092,192.168.1.252:9092		//kafka集群

  # buffer settings
  buffer_type file
  buffer_path /var/log/td-agent/buffer/td
  flush_interval 3s

  # topic settings
  default_topic wangrubintest		//kafka上的topic

  # data type settings
  output_data_type json
  compression_codec gzip

  # producer settings
  max_send_retries 1
  required_acks -1
</match>
{% endhighlight %}

{% highlight shell %}
mkdir plugins
mkdir -p /var/log/td-agent/buffer/td
{% endhighlight %}

* 构建Docker镜像

{% highlight shell %}
cd fluentd/
docker build -t 10.0.0.39:5000/fluentd:v10 .
{% endhighlight %}

* 启动fluent

{% highlight shell %}
docker run -d -p 24224:24224/tcp -p 24224:24224/udp --rm --name fluentd 10.0.0.39:5000/fluentd:v10
{% endhighlight %}

----------------------------------------------

## 部署elasticsearch

{% highlight shell %}
docker run -d --rm -p 9200:9200 --name elasticsearch hub.c.163.com/library/elasticsearch:2.4
{% endhighlight %}

----------------------------------------------

## 部署logstash

* 准备logstash.conf配置文件

{% highlight shell %}
mkdir -p /root/logstash && vim /root/logstash/logstash.conf
#

input {    
    kafka {    
        zk_connect => "192.168.1.250:2181,192.168.1.251:2181,192.168.1.252:2181"
        group_id => "es"    
        topic_id => "wangrubintest"
        codec => plain
        reset_beginning => false # boolean (optional)， default: false
        consumer_threads => 5  # number (optional)， default: 1
        decorate_events => false # boolean (optional)， default: false
    }
}

output {
     elasticsearch {         
        hosts => ["192.168.1.251:9200"]
         index => "logstash-%{+YYYY.MM.dd}"
        workers => 5
        flush_size => 3840
        idle_flush_time => 10
        template_overwrite => true
    }
    stdout{
        codec => rubydebug
    }
}
{% endhighlight %}


* 启动logstash

{% highlight shell %}
docker run -d --rm -v /root/logstash/:/config-dir/ logstash:2.4 -f /config-dir/logstash.conf
{% endhighlight %}

-------------------------------------------------------

## 部署kibana

{% highlight shell %}
ocker run -d --rm --name kibana -e ELASTICSEARCH_URL=http://192.168.1.251:9200 -p 5601:5601 hub.c.163.com/library/kibana:4.6
{% endhighlight %}


----------------------------------

## 环境检查

> http://192.168.1.250:5601/status

![1]({{ site.url }}/pic/FKLEK/1.png)


---------------------------------------------------------

## 部署nginx测试

* 下载镜像

{% highlight shell %}
docker pull 10.0.0.39:5000/nginx:v1.12
{% endhighlight %}

* 启动镜像

{% highlight shell %}
docker run -d -p 80:80 --rm --log-driver=fluentd --log-opt fluentd-address=fluentd.test.com:24224 --log-opt tag=nginx_access --name nginx 10.0.0.39:5000/nginx:v1.12
{% endhighlight %}

* 访问nginx并去kibana观察日志

![1]({{ site.url }}/pic/FKLEK/4.png)

![1]({{ site.url }}/pic/FKLEK/2.png)

![1]({{ site.url }}/pic/FKLEK/3.png)


---------------------------------------------------------------

Docker ==> Fluentd ==> Kafka ==> Logstash ==> ElasticSearch ==> Kibana 测试完毕

end






