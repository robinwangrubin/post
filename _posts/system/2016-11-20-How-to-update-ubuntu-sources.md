---
layout: post
title: Ubuntu 替换默认源
categories: System
---



* 备份默认源

{% highlight shell %}
cp /etc/apt/sources.list /etc/apt/sources.list.bak
{% endhighlight %}

* 选择合适的源，重定向到sources.list

使用官方源：

{% highlight shell %}
cat >/etc/apt/sources.list <<EOF
deb http://archive.ubuntu.com/ubuntu/ trusty main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu/ trusty-security main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu/ trusty-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu/ trusty-backports main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu/ trusty main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu/ trusty-security main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu/ trusty-updates main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu/ trusty-backports main restricted universe multiverse
EOF
{% endhighlight %}


使用163源：


{% highlight shell %}
cat >/etc/apt/sources.list <<EOF
deb http://mirrors.163.com/ubuntu/ trusty main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ trusty-security main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ trusty-updates main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ trusty-backports main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ trusty main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ trusty-security main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ trusty-updates main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ trusty-backports main restricted universe multiverse
EOF
{% endhighlight %}


使用搜狐源：



{% highlight shell %}
cat >/etc/apt/sources.list <<EOF
deb http://mirrors.sohu.com/ubuntu/ trusty main restricted universe multiverse
deb http://mirrors.sohu.com/ubuntu/ trusty-security main restricted universe multiverse
deb http://mirrors.sohu.com/ubuntu/ trusty-updates main restricted universe multiverse
deb http://mirrors.sohu.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb http://mirrors.sohu.com/ubuntu/ trusty-backports main restricted universe multiverse
deb-src http://mirrors.sohu.com/ubuntu/ trusty main restricted universe multiverse
deb-src http://mirrors.sohu.com/ubuntu/ trusty-security main restricted universe multiverse
deb-src http://mirrors.sohu.com/ubuntu/ trusty-updates main restricted universe multiverse
deb-src http://mirrors.sohu.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb-src http://mirrors.sohu.com/ubuntu/ trusty-backports main restricted universe multiverse
EOF
{% endhighlight %}

> Note：不同系统版本需参考下面更改源中的名称即可，例如：sed -i 's#trusty#xenial#g' sources.list

![1]({{ site.url }}/pic/update-ubuntu-sources/308450367.png)


![1]({{ site.url }}/pic/update-ubuntu-sources/859440133.png)


* 刷新源

{% highlight shell %}
sudo apt-get update
{% endhighlight %}

* 更新软件

{% highlight shell %}
sudo apt-get -y upgrade
{% endhighlight %}

* 清除本地缓存

{% highlight shell %}
apt-get clean
rm -rf /var/lib/apt/lists/*
{% endhighlight %}

------------------

end


