---
layout: post
title: "Docker Mesos集群架构"
categories: Docker
---


* content
{:toc}

## 实验环境准备

使用VMware创建5台虚拟机搭建mesos集群：

* 系统要求：CentOS-7.2
* 内核要求：3.10.0-327.el7.x86_64
* Docker版本：Docker version 1.12.3, build 6b644ec
* JAVA环境：JDK_1.8
* Mesos版本：mesos-1.1.0
* Zookeeper版本：zookeeper-3.4.6
* Marathon版本：marathon-1.3.6

主机名和IP地址规划如下：

mesos-master01 ==== 192.168.1.101	mesos-master zookeeper marathon

mesos-master02 ==== 192.168.1.102   mesos-master zookeeper

mesos-master03 ==== 192.168.1.103   mesos-master zookeeper

mesos-slave01 ==== 192.168.1.104	mesos-slave

mesos-slave02 ==== 192.168.1.105	mesos-slave

-------------------------

## 系统环境准备

* 主机名解析

所有节点：

{% highlight shell %}
cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.101  mesos-master01
192.168.1.102  mesos-master02
192.168.1.103  mesos-master03
192.168.1.104  mesos-slave01
192.168.1.105  mesos-slave02
{% endhighlight %}



* JAVA_1.8环境部署

master节点：

{% highlight shell %}
tar xf jdk-8u11-linux-x64.tar.gz -C /usr/local/
cat >> /etc/profile <<EOF
export JAVA_HOME=/usr/local/jdk1.8.0_11
export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar
export PATH=\$JAVA_HOME/bin:\$PATH
EOF
source /etc/profile
{% endhighlight %}

------------------------

## 部署zookeeper

### 安装zk

`三台master节点部署zookeeper`

{% highlight shell %}
cd /application/tools/
wget http://apache.fayea.com/zookeeper/zookeeper-3.4.6/zookeeper-3.4.6.tar.gz
tar xf zookeeper-3.4.6.tar.gz
mv zookeeper-3.4.6 /application/
ln -s /application/zookeeper-3.4.6/ /application/zookeeper
mv /application/zookeeper/conf/zoo_sample.cfg /application/zookeeper/conf/zoo.cfg
{% endhighlight %}

### 配置zk

{% highlight shell %}
grep '^[a-z]' zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
clientPort=2181
server.1=192.168.1.101:3181:4181
server.2=192.168.1.102:3181:4181
server.3=192.168.1.103:3181:4181
{% endhighlight %}


{% highlight shell %}
mkdir -p /tmp/zookeeper/
echo 1 > /tmp/zookeeper/myid		//master01执行
echo 2 > /tmp/zookeeper/myid		//master02执行
echo 3 > /tmp/zookeeper/myid		//master03执行
{% endhighlight %}

### 启动zk

{% highlight shell %}
/application/zookeeper/bin/zkServer.sh start /application/zookeeper/conf/zoo.cfg
{% endhighlight %} 

### 验证zk

{% highlight shell %}
[root@mesos-master01 ~]# /application/zookeeper/bin/zkServer.sh status /application/zookeeper/conf/zoo.cfg
JMX enabled by default
Using config: /application/zookeeper/conf/zoo.cfg
Mode: follower
{% endhighlight %}

{% highlight shell %}
[root@mesos-master02 ~]# /application/zookeeper/bin/zkServer.sh status /application/zookeeper/conf/zoo.cfg
JMX enabled by default
Using config: /application/zookeeper/conf/zoo.cfg
Mode: leader
{% endhighlight %}

{% highlight shell %}
[root@mesos-master03 ~]# /application/zookeeper/bin/zkServer.sh status /application/zookeeper/conf/zoo.cfg
JMX enabled by default
Using config: /application/zookeeper/conf/zoo.cfg
Mode: follower
{% endhighlight %}

zk部署完成

-------------------------

## 部署mesos

### 安装mesos

rpm包安装

> https://open.mesosphere.com/downloads/mesos/

`所有节点都部署mesos`

{% highlight shell %}
wget http://repos.mesosphere.com/el/7/x86_64/RPMS/mesos-1.1.0-2.0.107.centos701406.x86_64.rpm
yum localinstall mesos-1.1.0-2.0.107.centos701406.x86_64.rpm -y
{% endhighlight %}

Yum安装

{% highlight shell %}
rpm -ivh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
yum -y install mesos
{% endhighlight %}


### 配置mesos

`所有节点加入如下配置`

{% highlight shell %}
echo "zk://192.168.1.101:2181,192.168.1.102:2181,192.168.1.103:2181/mesos" > /etc/mesos/zk 
{% endhighlight %}

`master节点加入如下配置`

{% highlight shell %}
echo 2 >/etc/mesos-master/quorum
{% endhighlight %}

### 启动mesos-master

{% highlight shell %}
systemctl enable mesos-master
systemctl start mesos-master
{% endhighlight %}

### 启动mesos-slave

{% highlight shell %}
systemctl enable mesos-slave
systemctl start mesos-slave
{% endhighlight %}

### 验证mesos

http://192.168.1.101:5050

![1]({{ site.url }}/pic/docker-mesos/1.png)

![2]({{ site.url }}/pic/docker-mesos/2.png)


MASTER=$(mesos-resolve \`cat /etc/mesos/zk\`)

mesos-execute --master=$MASTER --name="cluster-test" --command="sleep 180"

![3]({{ site.url }}/pic/docker-mesos/3.png)



### 常见异常

![3]({{ site.url }}/pic/docker-mesos/7.png)

遇到上面的错误你一定是通过VMware或VirtualBox实验，

> 如果是在自己笔记本上通过虚拟机做实验，需要修改自己笔记本的hosts文件进行主机名解析；生产场景有DNS可忽略

`添加自己笔记本电脑的hosts文件解析`

{% highlight shell %}
192.168.1.101  mesos-master01
192.168.1.102  mesos-master02
192.168.1.103  mesos-master03
192.168.1.104  mesos-slave01
192.168.1.105  mesos-slave02
{% endhighlight %}



----------------------------

## 部署marathon

### 安装marathon

`第一台master上部署marathon`

{% highlight shell %}
rpm -ivh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
yum -y install marathon
{% endhighlight %}

`修改所有slave节点配置并重启slave`

{% highlight shell %}
echo 'docker,mesos' | tee /etc/mesos-slave/containerizers
systemctl start marathon
{% endhighlight %}


### 验证marathon

![4]({{ site.url }}/pic/docker-mesos/4.png)


http://192.168.1.101:8080

![5]({{ site.url }}/pic/docker-mesos/5.png)

-------------------------

## marathon调用mesos部署docker容器

vim /root/nginx.json

{% highlight shell %}
{
    "id": "nginx",
    "cpus": 0.2,
    "mem": 32,
    "instances": 1,
    "constraints": [],
    "container": {
        "type": "DOCKER",
        "docker": {
            "image": "nginx",
            "network": "BRIDGE",
            "portMappings": [
                {
                    "containerPort": 80,
                    "hostPort": 0,
                    "servicePort": 0,
                    "protocol": "tcp"
                }
            ]
        }
    }
}
{% endhighlight %}

{% highlight shell %}
curl -X POST http://192.168.1.101:8080/v2/apps -d @/root/nginx.json -H "Content-type: application/json"
{% endhighlight %}

![6]({{ site.url }}/pic/docker-mesos/6.png)

----------------------

end

