---
layout: post
title: "Docker 基础知识点介绍"
categories: Docker
---


* content
{:toc}

## Install docker 1.12

> Docker requires a 64-bit OS and version 3.10 or higher of the Linux kernel.

To check your current kernel version, open a terminal and use uname -r to display your kernel version:

{% highlight shell %}
uname -r
3.10.0-327.el7.x86_64
{% endhighlight %}

### Install with yum

Add the yum repo.

{% highlight shell %}
tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
{% endhighlight %}

Install the Docker package.

{% highlight shell %}
yum install docker-engine
{% endhighlight %}

Enable the service and start docker daemon.

{% highlight shell %}
systemctl enable docker.service
systemctl start docker
{% endhighlight %}

Verification

{% highlight shell %}
docker version
{% endhighlight %}

### Install Docker from binaries

Download the docker binaries

{% highlight shell %}
wget https://get.docker.com/builds/Linux/x86_64/docker-1.12.0.tgz
{% endhighlight %}

Install and start docker daemon

{% highlight shell %}
tar xf docker-1.12.0.tgz
mv docker/* /usr/bin/
dockerd &
{% endhighlight %}

Verification

{% highlight shell %}
docker version
{% endhighlight %}

### Install with the script

{% highlight shell %}
curl -fsSL https://get.docker.com/ | sh
{% endhighlight %}

{% highlight shell %}
systemctl enable docker.service
{% endhighlight %}

{% highlight shell %}
systemctl start docker
{% endhighlight %}

----------------

## Docker Command

* 下载镜像

> docker pull ubuntu:14.04

* 导出镜像

> docker save centos >/opt/centos.tar.gz

* 导入镜像

> docker load </opt/centos.tar.gz

* 列出当前已下载的镜像

> docker images

* 删除镜像

> docker rmi image-ID

ps:如果镜像被用来创建了容器，那么需先删除容器

* 在dockerhub上搜索镜像

> docker search centos

* 启动一个容器

> docker run -name myfirst_docker -it ubuntu:14.04 /bin/bash

* 后台启动容器

> docker run -d centos:6.6 /bin/bash -c “whilr ture;do echo “hahah”;sleep 1;done”

-d:后台启动

-name：定义容器名称

-i：打开容器的标准输入

-t：分配一个伪终端并绑定到容器的标准输入上

* 容器退出时自动删除该容器

> docker run –rm centos echo “hello world”

* 启动一个已停止的容器

> docker start container-ID

* 查看启动的容器

> docker ps

* 查看所有的容器

> docker ps -a

* 获取容器内的log

> docker logs container-ID

* 查看容器内部进程

> docker top container-ID

* 停止容器

> docker stop container-ID

* 极端方式停止容器（不推荐）

> docker kill container-ID

* 获取所有启动的容器的ID号

> docker ps -q

* 获取所有容器的ID号

> docker ps -a -q

* 批量杀掉启动的容器

> docker kill $(docker ps -q)

* 删除已停止的容器

> docker rm container-ID

* 删除正在运行的容器

> docker rm -f container-ID

* 删除所有容器和镜像

> docker kill $(docker ps -q)

> docker rm $(docker ps -a -q)

> docker rmi $(docker images -q -a)

* 进入正在运行的容器中

> docker attach container-ID

Note：多个窗口同时attach到同一个容器中时，会同步显示某个窗口的一切信息，当这个窗口阻塞，其他窗口也无法操作

* nsenter命令

nsenter可以访问另一个进程的名字空间，需要root权限

yum -y install util-linux

docker inspect –format "\{\{.State.Pid\}\}" 867e6627a194

找到容器的第一个进程的PID

nsenter -t 20012 -u -i -n -p

利用PID进入该容器

* commit镜像

> docker commit -m “my nginx” container-ID test/mynginx:v1 #自定义一个镜像

----------------

## Docker的4种网络模式

### host模式

> 众所周知，Docker使用了Linux的Namespaces技术来进行资源隔离，如PID Namespace隔离进程，Mount Namespace隔离文件系统，Network Namespace隔离网络等。一个Network Namespace提供了一份独立的网络环境，包括网卡、路由、Iptable规则等都与其他的Network Namespace隔离。一个Docker容器一般会分配一个独立的Network Namespace。但如果启动容器的时候使用host模式，那么这个容器将不会获得一个独立的Network Namespace，而是和宿主机共用一个Network Namespace。容器将不会虚拟出自己的网卡，配置自己的IP等，而是使用宿主机的IP和端口。 例如，我们在10.10.101.105/24的机器上用host模式启动一个含有web应用的Docker容器，监听tcp80端口。当我们在容器中执行任何类似ifconfig命令查看网络环境时，看到的都是宿主机上的信息。而外界访问容器中的应用，则直接使用10.10.101.105:80即可，不用任何NAT转换，就如直接跑在宿主机中一样。但是，容器的其他方面，如文件系统、进程列表等还是和宿主机隔离的。

### container模式

> 在理解了host模式后，这个模式也就好理解了。这个模式指定新创建的容器和已经存在的一个容器共享一个Network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的IP，而是和一个指定的容器共享IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过lo网卡设备通信。

### none模式

> 这个模式和前两个不同。在这种模式下，Docker容器拥有自己的Network Namespace，但是，并不为Docker容器进行任何网络配置。也就是说，这个Docker容器没有网卡、IP、路由等信息。需要我们自己为Docker容器添加网卡、配置IP等。

### bridge模式（默认模式）

> 当Docker server启动时，会在主机上创建一个名为docker0的虚拟网桥，此主机上启动的Docker容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。接下来就要为容器分配IP了，Docker会从RFC1918所定义的私有IP网段中，选择一个和宿主机不同的IP地址和子网分配给docker0，连接到docker0的容器就从这个子网中选择一个未占用的IP使用。如一般Docker会使用172.17.0.0/16这个网段，并将172.17.42.1/16分配给docker0网桥（在主机上使用ifconfig命令是可以看到docker0的，可以认为它是网桥的管理接口，在宿主机上作为一块虚拟网卡使用）。单机环境下的网络拓扑如下，主机地址为10.10.101.105/24。

* 启动容器时指定端口

> docker run -d -p 80:8080 nginx

host的80映射到container的8080端口

---------------------

## Docker存储

### 数据卷

-v /data挂载data目录 -v file 挂在一个文件 -v src:dst 指定一个挂载目录，开发常用，挂载物理机代码所在目录，nginx直接使用代码创建一个centos容器，挂载/data目录，挂载时可以指定权限 ro rw 等

{% highlight shell %}
docker run -it --name volume-test1 -v /data centos     #给容器挂载一个/data目录
docker inspect 023072ed38db |grep -A 5 "Mounts"  #查看容器中/data目录对应真实的物理位置
docker run -it -v /tmp:/opt centos   #挂载物理机的tmp目录到容器的opt下
{% endhighlight %}

### 数据卷容器

{% highlight shell %}
docker run -d --name myvolume centos
docker run -it --name myvolum-test --volumes-from myvolume centos
{% endhighlight %}

------------------------

## Dockerfile

Dockerfile:镜像的描述文件；所有镜像都由Dockerfile生成；

dockerfile 4大部分

* 基础镜像信息
* 维护者信息
* 镜像操作指令
* 容器启动时执行指令

Dockerfile 实例:

{% highlight shell %}
cat /opt/dockerfille/nginx/Dockerfile
# This is nginx Dockfile
# Verion 1.1.1
# Author Chuck.Ma
# Date 2016/1/6
FROM centos
MAINTAINER robin.wangrubin@gmail.com
RUN rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
RUN yum install -y nginx
RUN echo "Robin.Wangrubin" > /usr/share/nginx/html/index.html
RUN echo "daemon off;" >>/etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx"]
{% endhighlight %}

* 生成镜像文件

{% highlight shell %}
docker build -t test/mynginx:v5 /opt/dockerfille/nginx/
{% endhighlight %}

Note：/opt/dockerfille/nginx/下面的Dockerfile文件务必命名为[Dockerfile] ；D大写！

----------------------

## Namespace资源隔离

![1]({{ site.url }}/pic/docker-Basic/759122388670221.png)

### pid namespace

> 不同用户的进程就是通过pid namespace隔离开的，且不同 namespace 中可以有相同pid。所有的LXC进程在docker中的父进程为docker进程，每个lxc进程具有不同的namespace。同时由于允许嵌套，因此可以很方便的实现 Docker in Docker。

### net namespace

> 有了 pid namespace, 每个namespace中的pid能够相互隔离，但是网络端口还是共享host的端口。网络隔离是通过net namespace实现的， 每个net namespace有独立的 network devices, IP addresses, IP routing tables, /proc/net 目录。这样每个container的网络就能隔离开来。docker默认采用veth的方式将container中的虚拟网卡同host上的一个docker bridge: docker0连接在一起。

### ipc namespace

> container中进程交互还是采用linux常见的进程间交互方法(interprocess communication – IPC), 包括常见的信号量、消息队列和共享内存。然而同 VM 不同的是，container 的进程间交互实际上还是host上具有相同pid namespace中的进程间交互，因此需要在IPC资源申请时加入namespace信息 – 每个IPC资源有一个唯一的32位ID。

### mnt namespace

> 类似chroot，将一个进程放到一个特定的目录执行。mnt namespace允许不同namespace的进程看到的文件结构不同，这样每个 namespace 中的进程所看到的文件目录就被隔离开了。同chroot不同，每个namespace中的container在/proc/mounts的信息只包含所在namespace的mount point。

### uts namespace

> UTS(“UNIX Time-sharing System”) namespace允许每个container拥有独立的hostname和domain name, 使其在网络上可以被视作一个独立的节点而非Host上的一个进程。

### user namespace

> 每个container可以有不同的 user 和 group id, 也就是说可以在container内部用container内部的用户执行程序而非Host上的用户。

-------------------------

## Control Groups (cgroups)

cgroups 实现了对资源的配额和度量。 cgroups 的使用非常简单，提供类似文件的接口，在 /cgroup目录下新建一个文件夹即可新建一个group，在此文件夹中新建task文件，并将pid写入该文件，即可实现对该进程的资源控制。groups可以限制blkio、cpu、cpuacct、cpuset、devices、freezer、memory、net_cls、ns九大子系统的资源，以下是每个子系统的详细说明（可以使用docker run –help查看）：

* blkio 这个子系统设置限制每个块设备的输入输出控制。例如:磁盘，光盘以及usb等等。
* cpu 这个子系统使用调度程序为cgroup任务提供cpu的访问。
* cpuacct 产生cgroup任务的cpu资源报告。
* cpuset 如果是多核心的cpu，这个子系统会为cgroup任务分配单独的cpu和内存。
* devices 允许或拒绝cgroup任务对设备的访问。
* freezer 暂停和恢复cgroup任务。
* memory 设置每个cgroup的内存限制以及产生内存资源报告。
* net_cls 标记每个网络包以供cgroup方便使用。
* ns 名称空间子系统。

-------------------------

## Docker-Registry

Registry是docker的镜像仓库；主要用于存储企业内部私有的镜像文件，解决了镜像的安全性和高速拉取镜像问题。

### HTTP-registry

下载registry镜像

> docker pull registry:2

启动registry容器

> docker run -d -p 5000:5000 –restart=always –name registry -v /docker-registry/data:/var/lib/registry registry:2

retag镜像

> docker tag nginx 192.168.1.130:5000/nginx:v2

push镜像

> docker push 192.168.1.130:5000/nginx:v2

`The push refers to a repository [192.168.1.130:5000/nginx] Get https://192.168.1.130:5000/v1/_ping: http: server gave HTTP response to HTTPS client`

PS:目前registry默认需要以HTTPS方式访问；如果强制用HTTP访问，需要修改配置文件

`https://docs.docker.com/engine/admin/systemd/`

> mkdir /etc/systemd/system/docker.service.d

{% highlight shell %}
cat >/etc/systemd/system/docker.service.d/docker-config.conf<<EOF
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --insecure-registry=192.168.1.130:5000
EOF
{% endhighlight %}

> systemctl daemon-reload

> systemctl restart docker

### HTTPS-registry

Note: 有待补充

----------------------

## Docker-compose

docker-compose是一款应用控制工具，主要是针对application的；Dockerfile可以理解为镜像的描述文件，那么docker-compose的yaml文件可以理解为应用的描述文件。

假如：你有一个应用，需要一个lnmp环境，正常情况下你需要手动启动4个容器（linux-OS nginx mysql php），而且需要关联容器彼此之间的关系，以及需要考虑容器先后启动的顺序等；

但是有了docker-compose，你只需把该应用需要的环境信息写进一个文件即可，用一条命令即可启动整个应用；无需考虑容器之间的联系和启动顺序。

使用docker-compose一般有3个步骤

1. 定义应用需要的镜像的Dockerfile
2. 定义应用需要的服务（容器）
3. docker-compose up 启动应用

docker-compose.yml文件看起来像下面这样

* dockercoins应用

{% highlight shell %}
version: "2"
services:
  rng:
    build: rng
    ports:
    - "8001:80"

  hasher:
    build: hasher
    ports:
    - "8002:80"

  webui:
    build: webui
    ports:
    - "8000:80"
    volumes:
    - "./webui/files/:/files/"

  redis:
    image: redis

  worker:
    build: worker
{% endhighlight %}

* ELK应用

{% highlight shell %}
version: "2"

services:
  elasticsearch:
    image: elasticsearch
    # If you need to access ES directly, just uncomment those lines.
    #ports:
    #  - "9200:9200"
    #  - "9300:9300"

  logstash:
    image: logstash
    command: |
      -e '
      input {
        # Default port is 12201/udp
        gelf { }
        # This generates one test event per minute.
        # It is great for debugging, but you might
        # want to remove it in production.
        heartbeat { }
      }
      # The following filter is a hack!
      # The "de_dot" filter would be better, but it
      # is not pre-installed with logstash by default.
      filter {
        ruby {
          code => "
            event.to_hash.keys.each { |k| event[ k.gsub('"'.'"','"'_'"') ] = event.remove(k) if k.include?'"'.'"' }
          "
        }
      }
      output {
        elasticsearch {
          hosts => ["elasticsearch:9200"]
        }
        # This will output every message on stdout.
        # It is great when testing your setup, but in
        # production, it will probably cause problems;
        # either by filling up your disks, or worse,
        # by creating logging loops! BEWARE!
        stdout {
          codec => rubydebug
        }
      }'
    ports:
      - "12201:12201/udp"

  kibana:
    image: kibana
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
{% endhighlight %}

Compose has commands for managing the whole lifecycle of your application:

---------------

* Start, stop and rebuild services
* View the status of running services
* Stream the log output of running services
* Run a one-off command on a service

### Install docker-compose

> curl -L “https://github.com/docker/compose/releases/download/1.9.0/docker-compose-$(uname -s)-$(uname -m)” -o /usr/local/bin/docker-compose

> chmod +x /usr/local/bin/docker-compose

> curl -L https://raw.githubusercontent.com/docker/compose/$(docker-compose version –short)/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose

> docker-compose –version

------------------

### Get started with Docker Compose

下面我们利用docker-compose启动一个简单的python web程序

> Step 1: Setup

1.Create a directory for the project:

{% highlight shell %}
mkdir composetest
cd composetest
{% endhighlight %}


2.Create a file called `app.py` in your project directory and paste this in:

{% highlight shell %}
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host='redis', port=6379)

@app.route('/')
def hello():
    count = redis.incr('hits')
    return 'Hello World! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
{% endhighlight %}

3.Create another file called `requirements.txt` in your project directory and paste this in:


{% highlight shell %}
flask
redis
{% endhighlight %}

These define the application’s dependencies.

> Step 2: Create a Dockerfile

In this step, you write a Dockerfile that builds a Docker image. The image contains all the dependencies the Python application requires, including Python itself.

In your project directory, create a file named `Dockerfile` and paste the following:

{% highlight shell %}
FROM python:3.4-alpine
ADD . /code
WORKDIR /code
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
{% endhighlight %}

This tells Docker to:

* Build an image starting with the Python 3.4 image.
* Add the current directory `.` into the path `/code` in the image.
* Set the working directory to `/code`.
* Install the Python dependencies.
* Set the default command for the container to `python app.py`

> Step 3: Define services in a Compose file

Create a file called docker-compose.yml in your project directory and paste the following:

{% highlight shell %}
version: '2'
services:
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/code
  redis:
    image: "redis:alpine"
{% endhighlight %}

This Compose file defines two services, `web` and `redis`. The web service:

* Uses an image that’s built from the `Dockerfile` in the current directory.
* Forwards the exposed port 5000 on the container to port 5000 on the host machine.
* Mounts the project directory on the host to `/code` inside the container, allowing you to modify the code without having to rebuild the image.

The redis service uses a public `Redis` image pulled from the Docker Hub registry.

> Step 4: Build and run your app with Compose

1.From your project directory, start up your application.

{% highlight shell %}
docker-compose up
{% endhighlight %}

Compose pulls a Redis image, builds an image for your code, and start the services you defined.

2.Enter `http://0.0.0.0:5000/` in a browser to see the application running.

If you’re using Docker on Linux natively, then the web app should now be listening on port 5000 on your Docker daemon host. If `http://0.0.0.0:5000` doesn’t resolve, you can also try `http://localhost:5000`.

If you’re using Docker Machine on a Mac, use docker-machine ip MACHINE_VM to get the IP address of your Docker host. Then, open `http://MACHINE_VM_IP:5000` in a browser.

You should see a message in your browser saying:

`Hello World! I have been seen 1 times.`

3.Refresh the page. The number should increment.

> Step 5: Update the application

Because the application code is mounted into the container using a volume, you can make changes to its code and see the changes instantly, without having to rebuild the image.

1.Change the greeting in `app.py` and save it. For example:

{% highlight shell %}
return 'Hello from Docker! I have been seen {} times.\n'.format(count)
{% endhighlight %}

2.Refresh the app in your browser. The greeting should be updated, and the counter should still be incrementing.

> Step 6: Experiment with some other commands

If you want to run your services in the background, you can pass the `-d` flag (for “detached” mode) to `docker-compose up` and use `docker-compose ps` to see what is currently running:

{% highlight shell %}
docker-compose up -d
{% endhighlight %}

The `docker-compose run` command allows you to run one-off commands for your services. For example, to see what environment variables are available to the web service:

{% highlight shell %}
docker-compose run web env
{% endhighlight %}

See `docker-compose --help` to see other available commands. You can also install `command completion` for the bash and zsh shell, which will also show you available commands.

If you started Compose with `docker-compose up -d`, you’ll probably want to stop your services once you’ve finished with them:

{% highlight shell %}
docker-compose stop
{% endhighlight %}

You can bring everything down, removing the containers entirely, with the `down` command. Pass `--volumes` to also remove the data volume used by the Redis container:

{% highlight shell %}
docker-compose down --volumes
{% endhighlight %}

At this point, you have seen the basics of how Compose works.

------------------

end

----------------------

关于 Dockerfile 和 docker-compose 后面会有单独的章节讲解常用的命令参数以及更过示例.



