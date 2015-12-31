title: linux磁盘管理相关(施工中)
date: 2015-11-17 09:49:47
tags:
---

声明:
本博客欢迎转发
内容系本人学习, 研究和总结,有些是摘抄自互联网,如有雷同,实属荣幸.

## 文件系统

### ext4
linux kernel 自2.6.28 开始正式支持新的文件系统Ext4, 比前面的Ext3系统有很多增强,因此也是默认使用的文件系统.

### Btrfs:
Ceph-Osd在Btrfs上面有最佳的性能,支持Copy-On-Write, Writable snapshots. 问题是该文件系统显示还不是Production Ready, 不适合生产环境部署, 主要使用来作为测试.

### XFS
日志文件系统, 使Ceph系统中使用最多的文件系统,也是作为Ceph-OSD推荐使用的文件系统.

<!-- more-->

## linux 命令

### df
检查文件系统的磁盘占用情况,以及磁盘系统的挂载情况.
```
# df -hT           #以可读方式显示,并且列出磁盘文件系统类型
文件系统                           类型      容量  已用  可用 已用% 挂载点
/dev/mapper/ubuntu--kylin--vg-root ext4      443G  227G  194G   55% /
none                               tmpfs     4.0K     0  4.0K    0% /sys/fs/cgroup
udev                               devtmpfs  7.8G  4.0K  7.8G    1% /dev
tmpfs                              tmpfs     1.6G  1.2M  1.6G    1% /run
none                               tmpfs     5.0M     0  5.0M    0% /run/lock
none                               tmpfs     7.8G   43M  7.8G    1% /run/shm
none                               tmpfs     100M   52K  100M    1% /run/user
/dev/sda1                          ext2      236M   83M  142M   37% /boot
```

### fdisk
磁盘分区工具,不支持2TB以上的磁盘
```
# fdisk -l #查看硬盘信息

在终端输入 fdisk /dev/vdb
之后键入 m 可以看到帮助信息
键入 n 添加新分区
键入 p 选择添加主分区

之后,fdisk会让你选择分区的开始值和结束值
最后键入 w 保存所有并退出,完成新硬盘的分区

root@lrf-desktop-test:/home/ubuntu# fdisk -l

Disk /dev/vda: 42.9 GB, 42949672960 bytes
4 heads, 32 sectors/track, 655360 cylinders, total 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000c5852

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048    83886079    41942016   83  Linux

Disk /dev/vdb: 107.4 GB, 107374182400 bytes
16 heads, 63 sectors/track, 208050 cylinders, total 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x29dd845e

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048   104859647    52428800   83  Linux
/dev/vdb2       104859648   209715199    52427776   83  Linux

```

### parted

## LVM
LVM是Logical Volume Manager的简写, 它由Heinz Mauelshagen在Linux 2.4内核上实现。LVM将一个或多个硬盘的分区在逻辑上集合，相当于一个大硬盘来使用，当硬盘的空间不够使用的时候，可以继续将其它的硬盘的分区加入其中，这样可以实现磁盘空间的动态管理，相对于普通的磁盘分区有很大的灵活性。
