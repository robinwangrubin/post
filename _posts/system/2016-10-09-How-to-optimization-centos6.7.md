---
layout: post
title: 生产环境如何优化CentOS6.7
categories: System
---

* 查看系统基本信息

{% highlight shell %}
cat /etc/redhat-release|awk '{print "OPERATING SYSTEM:",$0}'         //查看系统发行版本
ifconfig eth0|awk -F "[ :]+" 'NR==2{print "IP-ADDRESS:",$4}'            //查看eth0网卡IP
uname -a|awk '{print "HOSTNAME:",$2"\nKERNEL VERSION:",$3}'             //查看系统内核信息
{% endhighlight %}

* 配置静态IP地址

{% highlight shell %}
vi /etc/sysconfig/network-scripts/ifcfg-eth0         //编辑网卡配置文件
DEVICE=eth0             //网卡名称
TYPE=Ethernet           //网卡类型
ONBOOT=yes              //是否开机启动
IPADDR=10.0.0.5         //IP地址
NETMASK=255.255.255.0           //子网掩码
GATEWAY=10.0.0.1                //默认网关
{% endhighlight %}


* 配置DNS

{% highlight shell %}
vi /etc/resolv.conf
nameserver 114.114.114.114              //主DNS地址
nameserver 8.8.8.8              //备DNS地址
{% endhighlight %}

* 创建普通用户

{% highlight shell %}
useradd test_admin
echo "password"|passwd --stdin test_admin
{% endhighlight %}


* 创建基本工作目录


{% highlight shell %}
mkdir -p /application/tools                  //下载的软件存放目录
mkdir -p /service/scripts/                              //脚本存放目录
{% endhighlight %}

* 设置开机启动项

{% highlight shell %}
chkconfig --list|grep 3:on|egrep -v "crond|network|rsyslog|sshd|sysstat"|awk '{print $1}'|sed -r 's#(.*)#chkconfig \1 off#g'|bash           //仅允许crond network rsyslog sshd sysstat 开机启动
{% endhighlight %}


* 关闭SElinux

{% highlight shell %}
sed -i 's/SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config 
{% endhighlight %}

* 开机自动清空网卡缓存信息

{% highlight shell %}
echo ">/etc/udev/rules.d/70-persistent-net.rules" >>/etc/rc.local
{% endhighlight %}


* 关闭IPtables防火墙

{% highlight shell %}
/etc/init.d/iptables stop
{% endhighlight %}

* 清空系统信息

{% highlight shell %}
>/etc/issue
{% endhighlight %}


* 设置别名

{% highlight shell %}
echo "alias egrep='egrep --color=auto'" >>/etc/profile
echo "alias grep='grep --color=auto'" >>/etc/profile
echo "alias vi='vim'" >>/etc/profile
{% endhighlight %}


* 更改默认yum源

{% highlight shell %}
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo
yum clean all
yum makecache
{% endhighlight %}

* 设置定时任务更新时间

{% highlight shell %}
touch /var/spool/cron/root
echo "*/5 * * * * /usr/sbin/ntpdate ntp1.aliyun.com >/dev/null 2>&1" >>/var/spool/cron/root
{% endhighlight %}


* 安装基本工具

{% highlight shell %}
yum install tree nmap sysstat lrzsz dos2unix -y
{% endhighlight %}

* SSH优化

{% highlight shell %}
Port 52113
listenAddress 10.0.0.10
PermitRootLogin no
PermitEmptyPasswords no
Use DNS no
GSSAPIAuthentication no
{% endhighlight %}


* 加大文件描述符

{% highlight shell %}
ulimit -SHn 65535
{% endhighlight %}


* 加密grub菜单

{% highlight shell %}
/sbin/grub-md5-crypt
{% endhighlight %}


* 禁止Linux系统被Ping

{% highlight shell %}
echo "net.ipv4.icmp_echo_ignore_all=1" >>/etc/sysctl.conf
sysctl -p
{% endhighlight %}

---------------

end

