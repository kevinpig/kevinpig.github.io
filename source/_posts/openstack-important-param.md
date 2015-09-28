layout: post
title: Openstack中的一些重要配置参数
date: 2015-09-24 09:20:45
tags:
 - Openstack
 - Devops
---

## 宿主资源隔离

> vcpu_pin_set = 4-$

虚拟机vCPU的绑定范围,可以防止虚拟机争抢宿主机进程的CPU资源,建议预留前几个物理CPU,把后面的所有CPU分配给虚拟机使用,可以配合cgroup或者内核启动参数来实现宿主进程不占用虚拟机使用的哪些CPU资源

> cpu_allocation_ratio = 4.0

物理CPU超售比例,默认是16倍,超线程也算作一个物理CPU

> ram_allocation_ratio = 1.0

内存分配超售比例,默认是1.5倍,生产环境不建议开启超售

> reserved_host_memory_mb = 4096

内存预留量,这部分内存不能被虚拟机使用

> reserverd_host_disk_mb = 10240

磁盘预留空间,这部分空间不能被虚拟机使用

> multi_host = True

是否开启nova-network的多节点模式,如果需要多节点部署,则该项需要设置为True

### 验证

校验虚拟机的vCPU亲和性,检查CPU是否成功隔离,创建一个虚机,然后通过libvirt参看

```
$ virsh list
 Id    Name                           State
 ----------------------------------------------------
  42    instance-00000553              running
  43    instance-00000596              running

$ virsh vcpuinfo 42
VCPU:           0
CPU:            4
State:          running
CPU time:       324210.2s
CPU Affinity:   ----yyyyyyyyyyyyyyyyyyyyyyyyyyyy
```

这就说明，虚拟机的vCPU对物理CPU的前四个核非亲和，他一般就会在后面的物理CPU中调度执行各种服务。
