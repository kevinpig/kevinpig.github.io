title: linux系统工具总结
date: 2015-11-04 11:29:30
tags:
 系统管理
---

声明:
本博客欢迎转发
内容系本人学习, 研究和总结,有些使摘抄自互联网,如有雷同,实属荣幸.

## 网络流量监控

### iftop

![iftop_interface](/images/linux_monitor/iftop-interface.jpg)

界面相关说明:

界面上面显示的是类似刻度尺的刻度范围，为显示流量图形的长条作标尺用的。

中间的<= =>这两个左右箭头，表示的是流量的方向。

TX：发送流量
RX：接收流量
TOTAL：总流量
Cumm：运行iftop到目前时间的总流量
peak：流量峰值
rates：分别表示过去 2s 10s 40s 的平均流量

<!-- more-->

```
# iftop -i eth0 -B #设定监测的网卡为eth0, 以byte为单位显示流量
```

```
高级功能: iftop 通过-f支持pcap-filter formatted filter的过滤条件设置.
```

### nethogs

iftop使按照目的IP地址和端口为视图来显示网络流量,如果想知道那个进程占用了大量的网络流量,就需要以进程为视图来显示网络流量.

```
# nethogs eth0
```

### netstat
用于显示各种网络相关信息,如网络连接, 路由表,接口状态. 从总体上看netstat输出分为两部分, 一为有源TCP连接,另一个为有源Unix域套接口.常用参数
1. -a (all)显示所有选项,默认不显示LISTEN相关
2. -t(tcp) 仅显示tcp相关选项
3. -l 仅列出又在Listen的服务状态
4. -p 显示建立相关链接的程序名
5. -u(udp) 仅显示udp相关选项

实例
```
# netstat -lt  #仅列出所有TCP监听端口
# netstat -pt  #在netstat输出中显示PID和进程名称
# netstat -anp | grep 80 # 查找端口xxx的进程名称
# netstat -lnp | grep 80 # 查找监听端口xxx的进程名称
# netstat -ap | grep ssh # 找出程序运行的端口
```

## 进程查看 & CPU & Mem

### htop

htop——一个可以让用户与之交互的进程查看器。作为文本模式的应用程序，主要用于控制台或 X 终端中。当前具有按树状方式来查看进程，支持颜色主题，可以定制等特性。

### atop

Atop 是一个类似 top 的工具，但比 top 更有料。通过 Atop，你能够监视 Linux 系统的性能状况，包括进程活动、CPU、内存、硬盘、网络等方面的使用情况等。

## 硬盘相关

### ncdu (磁盘目录占用空间计算排序工具)

ncdu命令是对传统du命令功能上的增强，不需要像du那样输入大量的命令，就可以计算文件及目录大小并可以按照大小或文件名进行排序

**常用快捷键**

n ：按文件名进行排序
s ：按文件大小进行排序
r ：重新统计当前文件夹大小
g ：用#或百分比显示各文件/目录的大小所占的百分比，例如：

![](/images/linux_monitor/ncdu-g.jpg)

i : 显示当前文件/目录信息，如下图

![](/images/linux_monitor/ncdu-i.jpg)
