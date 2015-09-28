layout: post
title: openstack中的可用域的配置
date: 2015-09-28 09:31:14
tags:
 - Openstack
---

## 前言
1. 什么使availablitiy\_zone
az 是用户可见的,用来手动的指定VM运行在哪些Host上. az 是在region范围内再次切分，例如可以吧一个机架上的机器划分在一个az中，划分az是为了提高容灾性和提供廉价的隔离服务。选择不同的region主要考虑那个region靠近你的用户群体。

2. 什么是aggregate host
host aggregate 是管理员用来根据硬件资源的某一属性对硬件进行划分的功能,只对管理员可见. 其主要功能就是实现根据某一属性来划分物理机,比如按照地理位置,使用固态硬盘的机器,内存超过32G的机器,根据这些指标来构成一个Host Group

在创建一个VM的时候,可以选择一个可用域,这样可以使创建的虚拟机位于可用域的Host机器上. 那么如何配置一个Host属于那个Availability Zone呢?

## G版本的实现

在G版本中Service表中不在保存Availability\_zone字段,配置项node\_available\_zone也不在使用.那么这时,如何确定一个host属于那个zone呢?

G版中,默认情况下,对Nova的服务分为两类,一类使Controller节点的服务进程,如nova-api, nova-scheduler, nova-conductor等,另一类使计算节点的进程, nova-computer. 对于第一类服务,默认的zone使配置项internal\_service\_availability\_zone, 而nova-computer所属的zone配置项default\_available\_zone决定.

<!-- more-->

社区的开发人员意识到,让管理员通过配置的方式管理zone不太合适,不够灵活.因此在G版中,创建一个Host-Aggregate可以指定一个availablity\_zone

```
[root@node-15 ~]# nova help aggregate-create
usage: nova aggregate-create <name> [<availability-zone>]

Create a new aggregate with the specified details.

Positional arguments:
  <name>               Name of aggregate.
  <availability-zone>  The availability zone of the aggregate (optional).

```

意思使创建一个host-aggregate,同时把它作为一个zone,此时aggregate=zone. 大家知道,aggregate使管理员可见,普通用户不可以见的对象,这个改变,可以使普通用户通过Zone的方式来使用Aggregate.

创建完aggregate之后,项aggregate中增加主机时,该主机就自动属于aggregate表示的zone.

因此，在G版，可以认为aggregate取代了zone，但同时又不影响aggregate与flavor的配合使用，因为这是两个调度层面。同时又要注意，一个主机可以加入多个aggregate中，所以G版中一个主机可以同时属于多个Availability Zone，这一点与F版也不同。

## 实际使用

1. 创建aggregate,制定zone
```
# nova aggregate-create aggregate_name az1
# nova aggregate-list
```
2. 添加主机

```
# nova aggregate-add-host aggregate_name computer1
```

3. 查询主机与服务所属的zone

```
# nova host-list
# nova service-list
```

4. 将一个主机加入另外一个aggregate,再次查询

```
# nova aggregate-create aggregate2 az2
# nova aggregate-add-host aggregate2 computer1
# nova host-
# nova service-list
```

## 参考链接
1. [F版和G版中的Availability Zone][F版和G版中的Availability Zone]

[F版和G版中的Availability Zone]: http://blog.csdn.net/lynn_kong/article/details/9012451 "Availability Zone"
