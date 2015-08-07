layout: post
title: 利用varant搭建跨平台开发环境
date: 2015-08-07 17:10:15
tags: vagrant
categories: dev
---

## 什么是vagrant
[vagrant](https://www.vagrantup.com)是一款构建虚拟开发环境的功能。可以使用vagrant封装一个标准的linux开发环境，然后分发给开发团队的成员。这样其他成员就可以在自己喜欢的桌面系统进行开发，而代码执行却可以在统一的环境，非常的便利。

## vagrant环境准备
1. 安装 [virtualBox](www.virtualbox.org)
2. 下载安装vagrant
3. 下载官方封装好的基础镜像 (比如ubuntu)
4. 添加镜像到vagrant ` $ vagrant box add trusty /path/to/trusty.box`

## 初始化开发环境
```
$ cd ~/dev
$ vagrant init trusty
$ vagrant up
```
看到终端显示启动完成以后，就可以通过ssh登录到虚拟机了
```
$ vagrant ssh 
$ cd /vagrant   # 切换到开发目录，这里的目录对应host机器上的～/dev目录
```

<!--more-->
## vagrant的功能
vagrant 初始化成功以后，会在初始化目录生产一个 `Vagrantfile`的配置文件，可以修改配置文件进行个性化定制

1. 共享host文件到VM中
默认情况下host会共享当前的文件夹到VM中的`/vargrant`下。 打开`Vagrantfile`,将下面这行注释去掉或者复制来增加新的共享文件 
` config.vm.sync_folder "../data", "/vagrant_data"`

2. 端口映射
vagrant默认使用端口映射方式将VM的端口映射到本地，从而实现类是`http://localhost:80`这种访问方式，更方便的方法是增加host-only网卡模式。打开`Vagrantfile`将下面这行注释去掉或者修改
`config.vm.network :private_network, ip: "192.168.33.10"`
重启虚拟机就是可以使用上面的IP地址进行访问VM了

3. privisioning -- 自动配置VM当使用`vagrant up`
可以通过shell脚本，或者puppet，chef，saltstack等配置管理工具，对vagrant开发环境进行自动配置。也可以使用`vagrant reload --provision`对已经运行过的VM进行快速的provision(将省略init中的import步骤)
使用shell脚本安装Apache。在Vagrant目录下创建booststrap.sh
``` 
 #!/usr/bin/env bash
 
 apt-get update
 apt-get install -y apache2
 if ! [ -L /var/www ]; then
    rm -rf /var/www
    ln -fs /vargant /var/www
 fi 
```
配置vagrant当setup开发机器的时候运行bootstrap脚本。修改`Vagrantfile`
```
 config.vm.provision :shell, path: "bootstrap.sh"
```
puppet, chef privision方法请参考`Vagrantfile`

## vagrant 常用命令
```
$ vagrant init              # inint VM
$ vagrant up                # start VM
$ vagrant halt              # stop VM
$ vagrant reload            # restart VM
$ vagrant ssh               # ssh login to VM
$ vagrant status            # view the VM status
$ vagrant destroy           # destroy the VM
```

## 参考链接
1. [http://segmentfault.com/a/1190000000264347](http://segmentfault.com/a/1190000000264347)
