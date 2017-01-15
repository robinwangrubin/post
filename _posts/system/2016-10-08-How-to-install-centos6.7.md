---
layout: post
title: "生产环境如何安装CentOS6.7"
categories: System
---

* content
{:toc}




## 下载CentOS-6.7镜像

[百度云链接：http://pan.baidu.com/s/1o7UNuno 密码:pvt6](http://pan.baidu.com/s/1o7UNuno)


----------------

* 选择：安装或升级现有操作系统

![1]({{ site.url }}/pic/how-to-install-centos6.7/01.png)


* 选择Skip跳过安装介质的检测


![2]({{ site.url }}/pic/how-to-install-centos6.7/02.png)

* 选择Next

![3]({{ site.url }}/pic/how-to-install-centos6.7/03.png)


* 选择安装系统时候的语言

![4]({{ site.url }}/pic/how-to-install-centos6.7/04.png)

* 选择键盘类型

![5]({{ site.url }}/pic/how-to-install-centos6.7/05.png)


* 选择存储设备类型

![6]({{ site.url }}/pic/how-to-install-centos6.7/06.png)


* 清除存储设备上所有数据

![7]({{ site.url }}/pic/how-to-install-centos6.7/07.png)


* 设置主机名

![8]({{ site.url }}/pic/how-to-install-centos6.7/08.png)


* 选择时区：亚洲上海并取消夏令时

![9]({{ site.url }}/pic/how-to-install-centos6.7/09.png)



* 设置系统root户名密码



![10]({{ site.url }}/pic/how-to-install-centos6.7/10.png)



* 如果密码不符合复杂性要求系统会有提示，选择Use Anyway

![11]({{ site.url }}/pic/how-to-install-centos6.7/11.png)


* 选择手动创建系统分区


![12]({{ site.url }}/pic/how-to-install-centos6.7/12.png)



* /boot :200M
* swap :物理内存的1.5-2倍
* / :剩余的容量全部给根分区


![13]({{ site.url }}/pic/how-to-install-centos6.7/13.png)


* 格式化硬盘

![14]({{ site.url }}/pic/how-to-install-centos6.7/14.png)



* 写入分区信息到硬盘

![15]({{ site.url }}/pic/how-to-install-centos6.7/15.png)


* 默认下一步即可

![16]({{ site.url }}/pic/how-to-install-centos6.7/16.png)


* 选择最小化安装和现在自定义安装软件包


![17]({{ site.url }}/pic/how-to-install-centos6.7/17.png)

* Base System: Base Compatibility libraries Debugging Tools
* Developemnt: Developemnt tools


![18]({{ site.url }}/pic/how-to-install-centos6.7/18.png)

![19]({{ site.url }}/pic/how-to-install-centos6.7/19.png)


* 重启系统即可

![20]({{ site.url }}/pic/how-to-install-centos6.7/20.png)


------------------------

end
