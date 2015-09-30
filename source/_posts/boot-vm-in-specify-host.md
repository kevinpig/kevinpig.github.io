layout: post<>
title: Openstack 决定虚机的宿主机
date: 2015-09-28 15:37:40
tags:
 - Openstack
---

声明:
本博客欢迎转发
内容系本人学习, 研究和总结,有些是摘抄自互联网,如有雷同,实属荣幸.

## 需求
通常在一个云计算中心中,提供虚拟资源池计算能力的物理主机一般有成百上千个,因为业务的需求,我们需要将这些物理主机进行分组,以区分其本身的性质或者提供不同的服务. 我们研究的使Openstack,因此这里就讨论Openstack中Nova如果通过配置将虚拟机创建到指定的Host或者分组中.

## 实现原理
当你发起一个创建虚机的请求,nova收到你的请求并分析所携带的创建约束条件,然后过滤出满足条件的主机,并在这些主机上创建虚机,这个过程主要有nova-scheduler 实现的. 最有主机的选择过程实际上就两个步骤,过滤和称重.

![主机过滤](http://docs.openstack.org/developer/nova/_images/filteringWorkflow1.png)

如上图,Filter Scheduler 首先得到未经过滤的主机列表,然后根据过滤属性,选择主机创建指定数目的虚拟机.

Nova中已经实现了多种的过滤策略,开发者也可以根据自己的需求来实现自己的过滤策略.

在/etc/nova/nova.conf中的scheduler\_default\_filter配置了起作用的filter

<!-- more -->

## 实际使用

在上一篇博客中[openstack中的可用域的配置](http://kevinpig.github.io/2015/09/28/openstack-availability-zone/) 我们阐述了如何创建available_zone,通过az我们可以使我们创建的虚机运行在指定的az上

```
# nova boot --image 1 --flavor 2 --availability-zone nova myServer
```

查看日志我们可以发现AvailabilityZoneFilter过滤出了az为nova的host

## 更近一步

当我们创建虚机的时候,如何单纯的通过指定flavor来确定虚机被创建的az呢? 我们可以使用AggregateInstanceExtraSpecsFilter来进行过滤,阅读源码我们可以得知,filter中的filter_properties 的组成大致如下
```
instance_type: 调用get_flavor_by_flavor_id和传入的flavor相关
availablity_zone: nova boot时传入的参数
```
get_flavor_by_flavor_id 返回的就是flavor的extra_specs属性，如此我们便可通过flavor来改变filter_properties，进而控制物理Host的选择策略。

创建一个新的flavor
```
[root@node-15 nova]# nova flavor-create test auto 1024 1 1
+--------------------------------------+------+-----------+------+-----------+------+-------+-------------+-----------+
| ID                                   | Name | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+--------------------------------------+------+-----------+------+-----------+------+-------+-------------+-----------+
| 900b9fd9-6686-4fec-9a12-166ce9184126 | test | 1024      | 1    | 0         |      | 1     | 1.0         | True      |
+--------------------------------------+------+-----------+------+-----------+------+-------+-------------+-----------+

```
给extra_specs添加一个availability_zone的metadata
```
[root@node-15 nova]# nova flavor-key 900b9fd9-6686-4fec-9a12-166ce9184126 set aggregate_instance_extra_specs:availability_zone=nova
[root@node-15 nova]# nova flavor-show 900b9fd9-6686-4fec-9a12-166ce9184126
+----------------------------+--------------------------------------------------------------+
| Property                   | Value                                                        |
+----------------------------+--------------------------------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                                        |
| OS-FLV-EXT-DATA:ephemeral  | 0                                                            |
| disk                       | 1                                                            |
| extra_specs                | {"aggregate_instance_extra_specs:availability_zone": "nova"} |
| id                         | 900b9fd9-6686-4fec-9a12-166ce9184126                         |
| name                       | test                                                         |
| os-flavor-access:is_public | True                                                         |
| ram                        | 1024                                                         |
| rxtx_factor                | 1.0                                                          |
| swap                       |                                                              |
| vcpus                      | 1                                                            |
+----------------------------+--------------------------------------------------------------+

```
修改nova.conf中的scheduler_default_filter配置,增加AggregateInstanceExtraSpecsFilter

修改完成后重启nova-scheduler服务,然后可以选用这个flavor创建虚机,最终你会看到这个flavor创建的虚拟都运行在指定的az中

## 参考链接

1. [Openstack Filter Scheduler][Openstack Filter Scheduler]
2. [巧用flavor metadata决定虚机的宿主机][flavor_metadata]

[Openstack Filter Scheduler]: http://docs.openstack.org/developer/nova/devref/filter_scheduler.html "Filter Scheduler"
[flavor_metadata]: http://niusmallnan.github.io/_build/html/_templates/flavor_metadata.html "flavor_metadata"
