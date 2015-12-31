layout: post
title: 搭建公司Openstack私有云环境
date: 2015-09-10 10:26:48
tags:
 - Openstack
categories:
 - 云计算
description:
---

## 1. 前言

　　随着IT技术的发展，云计算以不可逆转的方式掀起了新的IT产业变革，相比较传统的IT的计算，云计算有以下的巨大优势

 1. 可以使用大量的商用机器代替价格昂贵的大型机器，大大降低IT基础建设成本
 2. 云的分布式架构提供了更高的弹性和扩展性
 3. 虚拟化大大提高了IT管理的效率，降低的IT管理的难度

## 2. 目的

 1. 提高公司IT资产的利用率，提高IT的管理效率。使更多的人可以快速获取所需要的资源。
 2. 提供研发云平台，促进公司产品与云平台的兼容，帮助联影快速的构建所有类型的应用，包括移动应用，Web应用和Restful API
 3. 基于云平台，提供基于Hadoop和spark大数据计算平台，为公司大数据提供基础设施的支持平台。

<!--more-->

## 3. 技术选型

### 3.1 为什么选择openstack

  1. Openstack是现在最好的开源云平台
  2. Openstack有一个巨大的社区，以及众多的IT巨头（Redhat，Google，VMWare，Inter，Mirantis etc.)的支持
  3. 从长远来看，可以避免被任何一家私有供应商锁定。
  4. 中国也有许多优秀的公司(UnitedStack, EasyStack)提供托管云的服务，以及大量的研究Openstack的技术人员。可以方便获取商业的技术支持和招聘相关的技术人才。

### 3.2 为什么选择Ceph
　　Ceph是一种为优秀的性能、可靠性和可扩展性而设计的统一的、分布式的存储系统。Ceph可以一套系统同时提供块存储，对象存储，文件存储三种功能， 以便在满足不同应用需求的前提下简化部署和运维。而Ceph的分布式架构，则意味着真正的无中心架构设计和理论上没有上限的系统规模的可扩展性。同时Ceph也是一个优秀的**开源**系统。
　　在Openstack的IaaS系统中，涉及到存储部分主要为块存储服务（Cinder），对象存储服务（Swift），镜像管理模块（glance)以及计算服务（Nova）。
　　
![无统一存储](http://42.62.73.30/wordpress/wp-content/uploads/2013/09/QQ20130923-7.png)

![统一存储](http://42.62.73.30/wordpress/wp-content/uploads/2013/09/QQ20130923-8.png)

　　从上面的图可以看出当Openstack存储池使用了Ceph以后，所有的存储资源比有效的利用了起来。所有的镜像存储由Glance存储在Ceph中，所有的持久块设备由Cinder在Ceph中管理。当一个虚拟机启动时，Nova可以直接从Ceph中取得镜像然后启动，中间除了VM需要从Ceph中获得启动镜像内容以外不会有多余的传输消耗。
　　另外因为使用了共享存储，VM具备了Live Migration的能力，使IaaS具备了不关闭VM的情况下，将VM从一台Host主机迁移到另外一台Host机器的能力。

### 3.3 为什么选择Fuel
　　Openstack具有众多的组件，每个组件有需要的配置参数。手动搭建Openstack环境不仅效率低下，而且容易出错。Fuel部署工具使部署Openstack通过点击UI就可以完成操作系统的安装，网络的配置以及Openstack的配置，同时Fuel的Plugin机制使得用户可以根据自己的需要扩展其功能，满足自己的需要。同时Fuel使用的Mirantis Openstack具有完备的Log，Monitor And Alerting的解决方案，大大降低了搭建一个高可用Openstack的技术难度。
　　
## 4. 方案设计

### 4.1 Openstack网络原理
　　网络作为Openstack中最复杂组件，同时也是搭建和运维Openstack基础平台的最难的一部分。Openstack中可以选择的有Nova-Network和Neutron两种网络技术，从技术发展的角度来讲Neutron是趋势，而且可以实现自定义路由，LBaaS（Load Balance as a Service), FWaas(Fire wall as a Service) 等高级的功能，但是其也有本身的一些劣势（难以维护，Overlay网络性能低下），因此需要根据具体的业务需求来选择具体的技术。

#### 1. Nova-Network

　　Nova-network提供两种模式 FLATDHCP Manager 和VLan Manager，在Nova-network的模式下每个计算节点作为其承载的VM的网关，提供了一个更加平衡的网络解决方案，同时当一台计算节点down机后，其他节点的网络功能不受到影响。

+ FlatDHCP Manager

这种模式下所有的租户挂载在同一个L2网络下，也就是说VM之间没有二层隔离。

![](https://docs.mirantis.com/openstack/fuel/fuel-6.1/_images/flatdhcpmanager-sh_scheme.jpg)

+ Network VLan-Manager
Network vlan-manager更适合于比较大的云计算环境。它将属于不同租户的VM分组成不同的Vlan，这样就容许属于同一租户(project)的VM相互通信，而无法和其他租户的VM进行通信。为了支持这种方式，交换机的端口需要配置成trunk模式。

![](https://docs.mirantis.com/openstack/fuel/fuel-6.1/_images/vlanmanager_scheme.jpg)

但是Nova-Network的缺点也是明显的
 1. VLan技术固有缺陷，一个Region无法服务过多的租户
 2. 路由实现粗糙，路由决策和NAT依赖IP地址，很难实现Overlay IP，用户的IP管理不自由
 3. 提供的网络高级功能有限，自由基本的安全组。

#### 2. Neutron
　　Neutron 然后通过 UDP 等三层传输协议承载二层，形成 L2 over L3 的模型，这样我们就可以实现突破物理拓扑的任意自定义网络拓扑、Overlap IP 等.
　　
![](https://www.ustack.com/wp-content/uploads/2015/07/2.jpg)

但是最大的问题是网络节点单点故障和性能的问题。

![](http://www.sxt.cn/editor/attached/image/20140911/c597f8ff-0b0e-4535-bc9b-39e8fba6ff5d.jpg)

从图中可以明显看到东西向和南北向的流量会集中到网络节点，这会使网络节点成为瓶颈。

　　为了解决以上问题，社区在Juno版本上实现了一个分布式路由（DVR），就是将L3 Agent从单独的部署在网络节点转为分布式部署在所有的计算节点上，是计算节点可以像Nova-Network一样都有自己的Gateway。启用了DVR，情况会变为
![](http://www.sxt.cn/editor/attached/image/20140911/acd85aa9-94f5-4e30-ad31-3617d9679247.jpg)
对于东西向的流量， 流量会直接在计算节点之间传递。
对于南北向的流量，如果有floating ip，流量就直接走计算节点。如果没有floating ip，则会走网络节点。
　　看上去很美好，当然DVR也有自己的问题（当然我也看不懂，解释不清楚）。简单来说就是还不稳定，需要生产环境验证。（Fuel部署工具还不支持，在7.0版本会支持DVR方式的部署）
　　
#### 3. SDN
　　软件定义网络，这是更加高级的话题了，也是以后我们需要探索的方向。

----------

　　上面讲了那么多的废(ji)话(shu),具体到我们的项目中应该如何选择呢？具体场景具体分析，对于研发云来说，我们需要尝试更多的高级功能(LBaaS),选择Neutron作为我们的初始方案，经过测试运维后逐渐过渡到DVR模式。作为生产云，需要交付IT进行使用后就能保证稳定的运行，同时作为私有云不需要太多的租户隔离，因此**建议**使用Nova-network With VLan-Manager。
　　
### 4.2 网络规划
　　为了获得更好的网络性能和便于管理，我们将不同的流量划分到不同的逻辑网络上。

#### 1. Cluster管理网络（admin-PXE Network）
　　Fuel主节点使用这个网络发现和安装Openstack环境。Fuel主节点作为改DHCP服务器，因此该网络需要和其他网络隔离并且不能有其他的DHCP服务。Fuel节点通过该网络发现Host，通过PXE安装OS，进行OS网络配置以及Openstack组件的安装。

#### 2. 公网(Public Network)
　　这里的公网即指公司的网络，通过带网络我们可以访问Opentack Service以及我们搭建Openstack的Host机器。同时该网络地址池也作为VM的floating ip pool。当VM获取公网IP后，我们就可以通过我们的办公电脑和VM进行通信。
　　
　　如果使用Neutron网络模型，因为南北方向的流量需要经过网络节点，因此需要有限考虑将网络节点该网络升级为万兆接入公司万兆核心网。

#### 3. 存储网络 (万兆敏感）
　　作为集群的内部网络之一（即外界无法访问），主要是作为Ceph分布式存储系统内部流量的网络。

#### 4. Openstack管理网络（Management Network）（万兆敏感）
　　作为集群的内部网络，主要是为计算节点通过iSCSI协议来访问存储节点。同时也作为Openstack的内部网络来承载数据库查询，消息队列和其他信息。

#### 5. 私网(Private network)
　　私有网路用来作为VM相互通信的网络，即没有获取Floating IP地址的VM相互通信。因此此网段不属于公司网络地址空间。
　　
　　对于研发云，我们使用Neutron with VxLan，网络配置如下：
　　
　　
![](https://docs.mirantis.com/openstack/fuel/fuel-6.1/_images/Neutron_32_vlan_v2.png)


　　对于生产云，我们使用Nova-Network with VLan Manager:
　　
![](https://docs.mirantis.com/openstack/fuel/fuel-6.1/_images/preinstall_d_vlan.jpg)
　　
### 5. 运维
　　由于开源的好处，现在任何公司和个人都可以方便的获取云计算的代码进行部署和研究，同时部署工具的发展使的部署一个云计算环境也变的越来越傻瓜化。但是能运维好一个云环境还是需要很多的技术的积累。完成云计算环境的搭建对于我们来讲只是万里长征第一步。
　　

[1]: http://www.infoq.com/cn/news/2015/02/wal-mart-choose-openstack
