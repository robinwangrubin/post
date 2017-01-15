---
layout: post
title: "LNMP环境安装Zabbix3"
categories: System
---

* content
{:toc}

## Environment

* CentOS Linux release 7.2.1511 (Core)
* nginx:1.9.15
* Mysql:5.5.49
* PHP:5.5.35
* zabbix:3.2.3

----------------------

## Downloads Packages

{% highlight shell %}
mkdir /application/tools
cd /application/tools/
wget http://nginx.org/download/nginx-1.9.15.tar.gz
wget http://cn2.php.net/get/php-5.5.35.tar.gz/from/this/mirror && mv mirror php-5.5.35.tar.gz
wget http://dev.mysql.com/get/Downloads/MySQL-5.5/mysql-5.5.49.tar.gz
wget http://jaist.dl.sourceforge.net/project/zabbix/ZABBIX%20Latest%20Stable/3.2.3/zabbix-3.2.3.tar.gz
{% endhighlight %}

------------------------------

## Install Dependent

{% highlight shell %}
yum -y install gcc gcc-c++ autoconf automake zlib zlib-devel openssl openssl-devel pcre* make gd-devel libjpeg-devel libpng-devel libxml2-devel bzip2-devel libcurl-devel freetype-devel
{% endhighlight %}

----------------------------

## Install Nginx

{% highlight shell %}
useradd nginx -s /sbin/nologin -M
cd /application/tools/
tar xf nginx-1.9.15.tar.gz
cd nginx-1.9.15
./configure --prefix=/application/nginx-1.9.15 --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module
make && make install
ln -s /application/nginx-1.9.15/ /application/nginx
{% endhighlight %}

---------------------------

## Install PHP

{% highlight shell %}
cd /application/tools/
tar xf php-5.5.35.tar.gz 
cd php-5.5.35
./configure --prefix=/application/php-5.5.35 \
--with-config-file-path=/application/php-5.5.35/etc \
--with-bz2 \
--with-curl \
--enable-ftp \
--enable-sockets \
--disable-ipv6 \
--with-gd \
--with-jpeg-dir=/usr/local \
--with-png-dir=/usr/local \
--with-freetype-dir=/usr/local \
--enable-gd-native-ttf \
--with-iconv-dir=/usr/local \
--enable-mbstring \
--enable-calendar \
--with-gettext \
--with-libxml-dir=/usr/local \
--with-zlib \
--with-pdo-mysql=mysqlnd \
--with-mysqli=mysqlnd \
--with-mysql=mysqlnd \
--enable-dom \
--enable-xml \
--enable-fpm \
--with-libdir=lib64 \
--enable-bcmath
make && make install
ln -s /application/php-5.5.35/ /application/php
{% endhighlight %}

### Modify config

{% highlight shell %}
cd /application/tools/php-5.5.35
cp php.ini-production /application/php/etc/php.ini
cd /application/php/etc/
cp php-fpm.conf.default php-fpm.conf
sed -i 's#max_execution_time =.*#max_execution_time = 300#g' php.ini
sed -i 's#memory_limit = .*#memory_limit = 128M#g' php.ini
sed -i 's#post_max_size = .*#post_max_size = 16M#g' php.ini            
sed -i 's#upload_max_filesize = .*#upload_max_filesize = 2M#g' php.ini             
sed -i 's#max_input_time = .*#max_input_time = 300#g' php.ini 
sed -i 's#date.timezone = .*#date.timezone = PRC#g' php.ini    
{% endhighlight %}

### Check config

{% highlight shell %}
grep "max_execution_time = " php.ini
grep "memory_limit = " php.ini 
grep "post_max_size =" php.ini
grep "upload_max_filesize =" php.ini 
grep "max_input_time =" php.ini 
grep "date.timezone =" php.ini
{% endhighlight %}

--------------------------------

## Install Mysql

{% highlight shell %}
cd /application/tools/
tar xf mysql-5.5.49.tar.gz 
useradd mysql -s /sbin/nologin -M
mkdir /data/mysql -p
chown -R mysql.mysql /data/mysql/
yum install -y cmake gcc* ncurses-devel
cd mysql-5.5.49
cmake -DCMAKE_INSTALL_PREFIX=/application/mysql5.5.49 -DDEFAULT_CHARSET=utf8 -DENABLED_LOCAL_INFILE=1 -DMYSQL_DATADIR=/data/mysql -DWITH_EXTRA_CHARSETS=all -DWITH_READLINE=1 -DWITH_INNOBASE_STORAGE_ENGINE=1 -DMYSQL_TCP_PORT=3306 -DDEFAULT_COLLATION=utf8_general_ci
make && make install
ln -s /application/mysql5.5.49/ /application/mysql
\cp /application/mysql/bin/* /usr/bin/
{% endhighlight %}


### Modify config

{% highlight shell %}
mkdir -p /data/mysql
cd /application/mysql/support-files/
rm -f /etc/my.cnf
cp my-medium.cnf /data/mysql/my.cnf
cp mysql.server /etc/init.d/mysqld
chmod 755 /etc/init.d/mysqld 
{% endhighlight %}

--------------

## Install Zabbix

{% highlight shell %}
cd /home/mtime/tools/
tar xf zabbix-3.2.3.tar.gz
cd zabbix-3.2.3
yum install -y net-snmp-devel mysql-devel
useradd zabbix -s /sbin/nologin -M
./configure --prefix=/application/zabbix-3.0.3/ --enable-server --enable-agent --with-mysql --with-net-snmp --with-libcurl --with-libxml2
make install
ln -s /application/zabbix-3.0.3/ /application/zabbix
{% endhighlight %}

### Initialization

{% highlight shell %}
mysql -e "create database zabbix default charset utf8;"
mysql -e "grant all on zabbix.* to zabbix@localhost identified by 'zabbix';"
mysql -uzabbix -pzabbix zabbix </application/tools/zabbix-3.2.3/database/mysql/schema.sql
mysql -uzabbix -pzabbix zabbix </application/tools/zabbix-3.2.3/database/mysql/images.sql
mysql -uzabbix -pzabbix zabbix </application/tools/zabbix-3.2.3/database/mysql/data.sql
{% endhighlight %}

### Config zabbix_server

{% highlight shell %}
cat > /application/zabbix/etc/zabbix_server.conf <<EOF
ListenPort=10051
LogFile=/tmp/zabbix_server.log
PidFile=/tmp/zabbix_server.pid
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix
DBSocket=/tmp/mysql.sock
DBPort=3306
Timeout=4
LogSlowQueries=3000
EOF
{% endhighlight %}

### Config zabbix_agentd

{% highlight shell %}
cat > /application/zabbix/etc/zabbix_agentd.conf <<EOF
PidFile=/tmp/zabbix_agentd.pid
LogFile=/tmp/zabbix_agentd.log
LogFileSize=0
EnableRemoteCommands=1
Server=127.0.0.1,192.168.3.134
ServerActive=127.0.0.1,192.168.3.134:10051
Hostname=192.168.3.134
BufferSend=10
Timeout=30
AllowRoot=1
UnsafeUserParameters=1
EOF
{% endhighlight %}

--------------------------

### config Nginx

{% highlight shell %}
cd /application/nginx
mkdir html/zabbix
cat >conf/nginx.conf<<EOF
user  nginx;
worker_processes  1;

#error_log  logs/error.log warning;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
    include extra/*.conf;

}
EOF
mkdir conf/extra
cat >conf/extra/zabbix.conf <<EOF
server {
listen 80;
server_name localhost;
access_log /var/log/zabbix/nginx_zabbix.log main;
root html/zabbix;

location / {
           index index.html index.php index.html;
        }

location ~ .*\.(php|php5)?$
            {
             fastcgi_pass  127.0.0.1:9000;
             fastcgi_index index.php;
             include fastcgi.conf;
           }
}
EOF
mkdir -p /var/log/zabbix
\cp -a /application/tools/zabbix-3.2.3/frontends/php/* /application/nginx/html/zabbix/
/application/nginx/sbin/nginx -t
{% endhighlight %}

-------------------------

## Start service

> /application/nginx/sbin/nginx

> /application/php/sbin/php-fpm

> /etc/init.d/mysqld start

> /application/zabbix/sbin/zabbix_server

> /application/zabbix/sbin/zabbix_agentd

-----------------------

## Pic

![1]({{ site.url }}/pic/zabbix-install/1.png)

![2]({{ site.url }}/pic/zabbix-install/QQ截图20170103220119.png)

![3]({{ site.url }}/pic/zabbix-install/QQ截图20170103220130.png)

![4]({{ site.url }}/pic/zabbix-install/QQ截图20170103220140.png)

![5]({{ site.url }}/pic/zabbix-install/QQ截图20170103220148.png)

![6]({{ site.url }}/pic/zabbix-install/QQ截图20170103220158.png)

![7]({{ site.url }}/pic/zabbix-install/QQ截图20170103220339.png)

![8]({{ site.url }}/pic/zabbix-install/QQ截图20170103220350.png)

--------------

end

