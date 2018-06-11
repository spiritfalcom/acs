# VPC下 Kubernetes 的网络地址段规划 {#concept_izq_sg4_vdb .concept}

在阿里云上创建 Kubernetes 集群时，通常情况下，可以选择自动创建专有网络，使用默认的网络地址即可。在某些复杂的场景下，需要您自主规划 ECS 地址、Kubernetes Pod 地址和 Service 地址。本文将介绍阿里云 VPC 环境下 Kubernetes 里各种地址的作用，以及地址段该如何规划。

## Kubernetes 网段基本概念 {#section_oy4_yg4_vdb .section}

首先来看几个和 IP 地址有关的概念。

**VPC 网段**

您在创建 VPC 选择的地址段。只能从 10.0.0.0/8、172.16.0.0/12、192.168.0.0/16 三者当中选择一个。

**交换机网段**

在 VPC 里创建交换机时指定的网段，必须是当前 VPC 网段的子集（可以跟 VPC 网段地址一样，但不能超过）。交换机下面的 ECS 所分配到的地址，就是从这个交换机地址段内获取的。一个 VPC 下，可以创建多个交换机，但交换机网段不能重叠。

VPC 网段结构如下图。

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/6927/4613_zh-CN.png)

**Pod 地址段**

Pod 是 Kubernetes 内的概念，每个 Pod 具有一个 IP 地址。在阿里云容器服务上创建 Kubernetes 集群时，可以指定Pod 的地址段，不能和 VPC 网段重叠。比如 VPC 网段用的是 172.16.0.0/12，Kubernetes 的 Pod 网段就不能使用 172.16.0.0/16，不能使用 172.17.0.0/16…，这些地址都涵盖在 172.16.0.0/12 里了。

**Service 地址段**

Service 也是 Kubernetes 内的概念，每个 Service 有自己的地址。同样，Service 地址段也不能和 VPC 地址段重合，而且 Service 地址段也不能和 Pod 地址段重合。Service 地址只在 Kubernetes 集群内使用，不能在集群外使用。

Kubernetes 网段和 VPC 网段关系如下图。

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/6927/4617_zh-CN.png)

## 如何选择地址段 {#section_hjk_dh4_vdb .section}

**单 VPC+单 Kubernetes 集群场景**

这是最简单的情形。VPC 地址在创建 VPC 的时候就已经确定，创建 Kubernetes 集群时，选择和当前 VPC 不一样的地址段就可以了。

**单 VPC+多 Kubernetes 集群场景**

一个 VPC 下创建多个 Kubernetes 集群。在默认的网络模式下（Flannel），Pod 的报文需要通过 VPC 路由转发，容器服务会自动在 VPC 路由上配置到每个 Pod 地址段的路由表。所有 Kubernetes 集群的 Pod 地址段不能重叠，但 Service 地址段可以重叠。

VPC 地址还是创建 VPC 的时候确定的，创建 Kubernetes 的时候，为每个 Kubernetes 集群选择一个不重叠的地址段，不仅不能和 VPC 地址重叠，也不和其他 Kubernetes Pod 段重叠。

需要注意的是，这种情况下 Kubernetes 集群部分互通，一个集群的 Pod 可以直接访问另外一个集群的 Pod 和 ECS，但不能访问另外一个集群的 Service。

**VPC 互联场景**

两个 VPC 网络互联的情况下，可以通过路由表配置哪些报文要发送到对端 VPC 里。以下面的场景为例，VPC 1 使用地址段 192.168.0.0/16，VPC 2 使用地址段 172.16.0.0/12，我们可以通过路由表，指定在 VPC 1 里把 目的地址为172.16.0.0/12 的报文都发送到 VPC 2。

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/6927/4614_zh-CN.png)

在这种情况下，VPC 1 里创建的 Kubernetes 集群，首先不能和 VPC 1 的地址段重叠，同时其地址段也不能和 VPC 2 的地址段重叠。在 VPC 2 上创建 Kubernetes 集群也类似。这个例子中，Kubernetes 集群 Pod 地址段可以选择 10.0.0.0/8 下的某个子段。

**Note:** 这里要特别关注“路由到 VPC 2” 的地址段，可以把这部分地址理解成已经占用的地址，Kubernetes集群不能和已经占用的地址重叠。

如果 VPC 2 里要访问 VPC 1 的 Kubernetes Pod，则需要在 VPC 2 里配置到 VPC1 Kubernetes集群pod地址 的路由。

**VPC 网络到 IDC 的场景**

和 VPC 互联场景类似，同样存在 VPC 里部分地址段路由到 IDC ，Kubernetes 集群的 Pod 地址就不能和这部分地址重叠。IDC 里如果需要访问 Kubernetes 里的 Pod 地址，同样需要在 IDC 端配置到专线 VBR 的路由表。

