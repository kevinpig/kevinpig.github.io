layout: post
title: openstack中的Qos功能
date: 2015-09-30 10:08:54
tags:
 - Openstack
 - Qos
---

声明:
本博客欢迎转发
内容系本人学习, 研究和总结,有些使摘抄自互联网,如有雷同,实属荣幸.

## 需求
在云计算数据中心,一台宿主机上面要跑几十台的VM,如果一台VM运行了大量IO,或者CPU密集的任务,如何保证其他的VM的服务质量呢? 这样我们就需要限制VM的能力,来保证Qos.

## 实现

### 1. CPU Qos

在Openstack Grizzly版本中已经继承了一个叫做 [Instance Resource Quota](https://wiki.openstack.org/wiki/InstanceResourceQuota)的功能.这个功能基于cgroup和tc实现了CPU,disk IO和network IO的限流功能.Openstack在设置Flavor的时候,可以将这些限流信息设定好.当启动虚机的时候,写入/var/lib/nova/instance/xxx/libvirt.xml中就可以了.

我们可以大致测试一下,CPU的限制目前支持一下参数:

```
quota:cpu_shares
quota:cpu_period
quota:cpu_quota
```

通过CLI创建Flavor,并设置flavor的extra specs, 当然也可以通过dashboard设置
```
[root@node-15 rabbitmq]# nova flavor-create cpu-quota-test 88 512 1 1
+----+----------------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name           | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+----------------+-----------+------+-----------+------+-------+-------------+-----------+
| 88 | cpu-quota-test | 512       | 1    | 0         |      | 1     | 1.0         | True      |
+----+----------------+-----------+------+-----------+------+-------+-------------+-----------+
[root@node-15 rabbitmq]# nova flavor-key 88 set quota:cpu_period=5000
[root@node-15 rabbitmq]# nova flavor-key 88 set quota:cpu_quota=2500
[root@node-15 rabbitmq]# nova flavor-show 88
+----------------------------+---------------------------------------------------------+
| Property                   | Value                                                   |
+----------------------------+---------------------------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                                   |
| OS-FLV-EXT-DATA:ephemeral  | 0                                                       |
| disk                       | 1                                                       |
| extra_specs                | {"quota:cpu_period": "5000", "quota:cpu_quota": "2500"} |
| id                         | 88                                                      |
| name                       | cpu-quota-test                                          |
| os-flavor-access:is_public | True                                                    |
| ram                        | 512                                                     |
| rxtx_factor                | 1.0                                                     |
| swap                       |                                                         |
| vcpus                      | 1                                                       |
+----------------------------+---------------------------------------------------------+

```

然后使用这个flavor启动虚拟机,在计算节点上查看cgroup如下:

```
[root@node-21 vcpu0]# pwd
/cgroup/cpu/machine/instance-0000013e.libvirt-qemu/vcpu0
[root@node-21 vcpu0]# cat cpu.cfs_period_us
5000
[root@node-21 vcpu0]# cat cpu.cfs_quota_us
2500

```
默认的cpu period 使100000,这里已经改成5000,说明起作用了

参看libvirt.xml
```
[root@node-21 7e52545a-5e37-4bab-8dd4-46c6b906436e]# pwd
/var/lib/nova/instances/7e52545a-5e37-4bab-8dd4-46c6b906436e

  <cputune>
    <quota>2500</quota>
    <period>5000</period>
  </cputune>

```
<!-- more -->

### 2. Disk Qos

通过以上方法设置了Disk IO的Qos参数,,在windows下面使用iometer测试,发现并不能正常的起作用.想想我们搭建的Openstack中使用了Ceph作为统一存储,所有的IO操作都是通过网络写入分布式文件系统中的. 这时候我们就的用另外的方法来保证Disk 的Qos

Disk Qos的参数包含
```
total_bytes_sec
read_bytes_sec
write_bytes_sec
total_iops_sec
read_iops_sec
write_iops_sec
```

测试

1. 首先创建在Cinder中创建一个Qos
2. 创建一个Volume Type
3. 然后将Qos关联到Volume type上
```
[root@node-15 ~]# cinder qos-create high-iops consumer="front-end" read_iops_sec=2000 write_iops_sec=1000
+----------+---------------------------------------------------------+
| Property |                          Value                          |
+----------+---------------------------------------------------------+
| consumer |                        front-end                        |
|    id    |           48fb2aa2-8f80-4045-b6cc-a051292986cc          |
|   name   |                        high-iops                        |
|  specs   | {u'write_iops_sec': u'1000', u'read_iops_sec': u'2000'} |
+----------+---------------------------------------------------------+
[root@node-15 ~]# cinder type-create high-iops
+--------------------------------------+-----------+
|                  ID                  |    Name   |
+--------------------------------------+-----------+
| 7b115bf5-b217-479d-8821-e9201965c756 | high-iops |
+--------------------------------------+-----------+
[root@node-15 ~]# cinder qos-associate 48fb2aa2-8f80-4045-b6cc-a051292986cc  7b115bf5-b217-479d-8821-e9201965c756

```

完成后我们创建一个云硬盘,并将其attach到一个windows的虚机上. 使用iometer测试4k random 100% read发现最高IOPS为2000,使我们设置的Qos值,而当没有设置该Qos的时候,我们的虚机的IOPS可以达到20000.


## 参看链接
1. [Openstack Ceph rbd and Qos][Openstack_Ceph_Qos]
2. [Openstack CPU/Disk/network QoS 功能][Openstack_CPU/disk/network_Qos]
3. [Openstack中的网络QoS功能][Openstack_network_Qos]

[Openstack_Ceph_Qos]: http://ceph.com/planet/openstack-ceph-rbd-and-qos/ "Ceph Rbd Qos"

[Openstack_CPU/disk/network_Qos]: http://blog.csdn.net/matt_mao/article/details/18356801 "Openstack CPU/Disk/network QoS 功能"

[Openstack_network_Qos]: http://www.xuebuyuan.com/2036978.html "Openstack中的网络QoS功能"
