---
layout: post
title: dmidecode
categories: Command
---


### 查看BIOS信息

> dmidecode -t 0

### 查看系统信息

> dmidecode -t 1

### 查看服务器u数电源数

> dmidecode -t 3

### 查看cpu信息

> dmidecode -t 4 \|grep Version
 
> dmidecode -t 7

### 查看服务器物理接口信息

> dmidecode -t 8

### 查看最大支持内存

> dmidecode -t 16

### 查看内存详细信息

> dmidecode -t 17

### 查看服务器电源信息

> dmidecode -t 39


