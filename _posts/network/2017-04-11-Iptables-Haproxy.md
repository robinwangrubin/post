---
layout: post
title: Iptables和HAproxy实现端口转发
categories: Network
---

* content
{:toc}


## 需求：

client访问公网IP-A+prot 映射到公网IP-B+port 


## 问题：

client访问的是A 数据包经过A之后目的变成B B收到数据包处理完毕回包，由于B会将数据包直接回向client；注意：client一开始访问的是A，回包的是B，所以三次握手都无法建立成功。


## 解决方案：

避免公网B直接给真实client回包；所以除了做DNAT外还需做SNAT

## 模拟测试：

访问192.168.1.11:8080 --->>202.205.184.149:80（中国教育网IP）

* 方案一：

192.168.1.11上配置iptables 做SNAT+DNAT

* 方案二：

192.168.1.11上配置haproxy实现TCP层反向代理负载均衡（单个realserver节点其实就是变相的SNAT+DNAT）

* 方案三：

192.168.1.11上配置LVS的Full NAT模式(SNAT+DNAT)

* 方案四：

192.168.1.11上配置nginx的4层转发(SNAT+DNAT)



## Iptables

* 开启路由转发，关闭系统sys防护（以便后面进行压测）

{% highlight shell %}
ulimit -n 65535
echo 1 > /proc/sys/net/ipv4/ip_forward
sed -i 's#net.ipv4.ip_forward = 0#net.ipv4.ip_forward = 1#g' /etc/sysctl.conf
sed -i 's#net.ipv4.tcp_syncookies = 1#net.ipv4.tcp_syncookies = 0#g' /etc/sysctl.conf
modprobe ip_tables
modprobe iptable_filter
modprobe iptable_nat
modprobe ip_conntrack
modprobe ip_conntrack_ftp
modprobe ip_nat_ftp
modprobe ipt_state
sysctl -p
{% endhighlight %}


{% highlight shell %}
/etc/init.d/iptables start
iptables -F
iptables -X
iptables -t nat -A PREROUTING -d 192.168.1.11 -p tcp --dport 8888 -j DNAT --to-destination 202.205.184.149:80
iptables -t nat -A POSTROUTING -d 202.205.184.149/32 -o eth0 -j SNAT --to-source 192.168.1.11
iptables-save >/etc/sysconfig/iptables
echo "/etc/init.d/iptables start" >> /etc/rc.local
{% endhighlight %}


![1]({{ site.url }}/pic/iptables-HAproxy/jiaoyu.png)


### 压力测试

> sed -i 's#net.ipv4.tcp_syncookies = 1#net.ipv4.tcp_syncookies = 0#g' /etc/sysctl.conf

`压测服务器 iptables 应用服务器都要关闭此选项才能进行高并发测试；否则报错：apr_socket_recv: Connection reset by peer (104)`

为了排除网络因素 因此我们不夸互联网进行测试


{% highlight shell %}
ulimit -n 65535
iptables -t nat -A PREROUTING -d 192.168.1.11 -p tcp --dport 8888 -j DNAT --to-destination 192.168.1.100:80
iptables -t nat -A POSTROUTING -d 192.168.1.100/32 -o eth0 -j SNAT --to-source 192.168.1.11
{% endhighlight %}

`192.168.1.100是我搭建的nginx web服务 提供了一个静态页面`


* 安装压力测试工具并且开始压测

{% highlight shell %}
yum install httpd-tools -y
ab -n 100000 -c 10000 http://192.168.1.11:8888/
{% endhighlight %}



![1]({{ site.url }}/pic/iptables-HAproxy/iptables01.png)

![1]({{ site.url }}/pic/iptables-HAproxy/iptables02.png)




## HAproxy

{% highlight shell %}
wget http://www.haproxy.org/download/1.7/src/haproxy-1.7.5.tar.gz
tar xf haproxy-1.7.5.tar.gz 
cd haproxy-1.7.5
uname -r
make TARGET=linux26 PREFIX=/usr/local/haproxy
make install PREFIX=/usr/local/haproxy
mkdir /etc/haproxy/conf -p
{% endhighlight %}




{% highlight shell %}
cat >> /etc/haproxy/conf/haproxy.cfg <<EOF
global  
        daemon  
        nbproc 10  
        pidfile /var/run/haproxy.pid  
defaults  
        mode tcp               #默认的模式mode { tcp|http|health }，tcp是4层，http是7层，health只会返回OK  
        retries 2               #两次连接失败就认为是服务器不可用，也可以通过后面设置  
        option abortonclose     #当服务器负载很高的时候，自动结束掉当前队列处理比较久的链接  
        maxconn 65535            #默认的最大连接数  
        timeout connect 5000ms  #连接超时  
        timeout client 30000ms  #客户端超时  
        timeout server 10000ms  #服务器超时  
        log 127.0.0.1 local0 err #[err warning info debug]  
listen test1  
        bind 0.0.0.0:8888  
        mode tcp  
        server s1 192.168.1.100:80
EOF
{% endhighlight %}


> /usr/local/haproxy/sbin/haproxy -f /etc/haproxy/conf/haproxy.cfg &


![1]({{ site.url }}/pic/iptables-HAproxy/ha01.png)

![1]({{ site.url }}/pic/iptables-HAproxy/ha02.png)


## 推测结论

HAproxy用来当做NAT设备使用，在高并发场景会消耗大量CPU计算资源，性能远不如iptables

Iptables做NAT设备使用几乎不消耗CPU和内存资源。



