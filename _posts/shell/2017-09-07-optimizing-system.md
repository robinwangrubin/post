---
layout: post
title: Centos6 & Centos 7 初始化脚本
categories: Shell
---

```
#!/usr/bin/env bash
#Author: xxxxxxx
#Email: xxxxxxx@gmail.com
#Description: This script is for optimizing system performance; For Centos6 and Centos7

cat /etc/redhat-release|awk '{print "OPERATING SYSTEM:",$0}'
grep "IPADDR" /etc/sysconfig/network-scripts/ifcfg-eth0|awk -F "=" '{print "HOST IPADDR:",$2}'
uname -a|awk '{print "HOSTNAME:",$2"\nKERNEL VERSION:",$3}'        

sed -i 's/DNS1=.*/DNS1=114.114.114.114/g' /etc/sysconfig/network-scripts/ifcfg-eth0
sed -i 's/DNS2=.*/DNS2=180.76.76.76/g' /etc/sysconfig/network-scripts/ifcfg-eth0
cat >/etc/resolv.conf<<EOF
nameserver 114.114.114.114
nameserver 180.76.76.76
EOF

sed -i 's/SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
setenforce 0 &> /dev/null

TEST=$(grep "soft nproc 65535" /etc/security/limits.conf|wc -l)
if [ "$TEST" == "0" ]; then
cat  >> /etc/security/limits.conf << EOF
* soft nproc 65535
* hard nproc 65535
* soft nofile 65535
* hard nofile 65535
EOF
fi

sed -i "s/\#UseDNS.*/UseDNS no/g" /etc/ssh/sshd_config
sed -i "s/GSSAPIAuthentication.*/GSSAPIAuthentication no/g" /etc/ssh/sshd_config

TEST=$(egrep "color|auto" /etc/profile|wc -l)
if [ $TEST -eq 0 ];then
echo "alias egrep='egrep --color=auto'" >>/etc/profile
echo "alias grep='grep --color=auto'" >>/etc/profile
echo "alias vi='vim'" >>/etc/profile
fi

TEST=$(grep "ntp1.aliyun.com" /var/spool/cron/root|wc -l)
if [ $TEST -eq 0 ];then
touch /var/spool/cron/root
echo "*/5 * * * * /usr/sbin/ntpdate ntp1.aliyun.com >/dev/null 2>&1" >>/var/spool/cron/root
fi

yum install wget -y &> /dev/null

SYSTEM_VERSION=$(cat /etc/redhat-release|grep -o "6.")
if [ "$SYSTEM_VERSION" == "" ];then
    VERSION='centos7'
else
    VERSION='centos6'
fi

if [ $VERSION == 'centos6' ];then
chkconfig --list|grep 3:on|egrep -v "crond|network|rsyslog|sshd|sysstat"|awk '{print $1}'|sed -r 's#(.*)#chkconfig \1 off#g'|bash &>/dev/null
/etc/init.d/iptables stop &> /dev/null
wget -qO /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo
wget -qO /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo
chmod +x /etc/rc.d/rc.local
elif [ $VERSION == 'centos7' ];then
wget -qO /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -qO /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all &> /dev/null
systemctl stop firewalld.service &> /dev/null
systemctl disable firewalld.service &> /dev/null
systemctl stop NetworkManager.service &> /dev/null
systemctl disable NetworkManager.service &> /dev/null
systemclt stop postfix.service &> /dev/null
systemclt disable postfix.service &> /dev/null
yum install iptables-services -y &> /dev/null
else 
echo "Pls Check The System Version"
fi
yum clean all &> /dev/null
yum makecache &> /dev/null
yum -y groupinstall Development tools
```

> wget -qO ./optimizing-system.sh http://wangrubin.com/scripts/optimizing-system.sh && sh optimizing-system.sh

---------------

end
