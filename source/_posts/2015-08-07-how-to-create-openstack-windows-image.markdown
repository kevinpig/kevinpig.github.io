---
layout: post
title: "如何创建openstack windows 虚拟机镜像"
date: 2015-08-07 10:57:39 +0800
comments: true
tags:
 - Openstack
 - windows
 - image
categories:
 - 云计算
 
description: 
---


## 前言

　　本文以Ubuntu 14.04为例来介绍如何创建windows的openstack虚拟机镜像。安装之前的装备工作包括
　　1. 准备一个windows 2008的iso文件
　　2. 下载virtio-win的驱动文件
　　3. 下载cloud-init的安装程序

## 1. 创建镜像磁盘

创建一个windows镜像磁盘，raw格式，大小为15G
``` 
$ qemu-img create -f qcow2 winServer2008.img 15G 
```

<!--more-->

## 2. 创建虚拟机

```
 $ virt-install --connect qemu:///system -n winServer2008 -r 2048 \
  --disk path=/home/lrf/imageCreate/winServer2008.img,size=15,format=qcow2,bus=virtio,cache=none \
  --cdrom=/home/lrf/cn_windows_server_2008_r2_with_sp1_x64_dvd_617598.iso \
  --os-type=windows --os-variant=win2k8 \
  --network network=default,model=virtio \
  --disk path=/home/lrf/virtio-win-0.1-100.iso,device=cdrom,perms=ro
```
　　以上参数比较多，不一一解释，详细请参看man virt-install。命令成功执行后提示‘域安装仍在进行，请等待完成安装’。这时候我们在Shell执行`virt-manager`来打开图形配置界面。
　　连接成功后，和普通安装操作系统没有区别。但在分区时需要加载驱动程序，选择Win7-AMD64即可。
　　![](http://static.oschina.net/uploads/space/2012/1212/203412_D0MJ_123777.jpg)
　　
　　
## 3. 安装cloud-init

![](http://www.cloudbase.it/wp-content/uploads/2015/07/cloudbase-init-01.png)

![](http://www.cloudbase.it/wp-content/uploads/2015/07/cloudbase-init-02.png)

![](http://www.cloudbase.it/wp-content/uploads/2015/07/cloudbase-init-03.png)

安装完成以后你可以找到名为Cloud Initialization Service的windows服务，该服务会在你下次重新启动windows的时候自动启动
![](http://www.cloudbase.it/wp-content/uploads/2012/12/CloudbaseInit_service.png)

安装完成Cloud-Init请执行Sysprep来生成一个通用的Image，这样我们就可以从该image复制多个新的实例。

## 4. 其他工作

 1. 在设备管理器中确认磁盘驱动器和网络驱动器使用的为redhat virto
 2. 为上传的镜像打开远程连接 (optional)
 3. 上传镜像完成后，设置安全策略，为远程连接开放3389端口
 ![](http://static.oschina.net/uploads/space/2012/1212/204846_9VEc_123777.jpg)
 
## 5. 上传虚拟机

　　通过Horizon的UI我们可以上传我们的镜像文件到Openstack中，接下来解释测试该镜像是否可以正确的启动虚拟机。
　　
作者：[kevinpig](ruifei.liu@united-imaging.com)

# 参考链接
1. [http://my.oschina.net/guol/blog/95449](http://my.oschina.net/guol/blog/95449)
2. [http://www.tuicool.com/articles/7ZR73q](http://www.tuicool.com/articles/7ZR73q)
3. [http://www.cloudbase.it/cloud-init-windows/](http://www.cloudbase.it/cloud-init-windows/)
