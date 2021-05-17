# 第一章：kubernetes基础

## Kubernetes介绍

Kubernetes是一个全新的基于容器的分布式架构领先方案，是谷歌Brog系统的一个开源版本，于2014年9月发布第一个版本，2015年7月发布第一个正式版本。

Kubernetes本质上是`` 一组服务器集群``，它可以在集群的每个节点上运行特定的程序，来对节点中部署的容器进行管理。

目的是``实现资源管理的自动化``。主要提供了如下的功能：

* **自我修复**：一旦某个容器崩溃，能够在1秒之内迅速启动新的容器
* **弹性伸缩**：可以根据需要，自动对集群中正在运行的容器数量进行调整
* **服务发现**：服务可以通过自动发现的形式找到它所依赖的服务
* **负载均衡**：如果一个服务启动了多个容器，能够自动实现请求的负载均衡
* **版本回退**：如果发现新版本的程序有问题，可以立即回退到原来的版本
* **存储编排**：可以根据容器自身需求自动创建存储卷

![](/Users/tanglongan/Documents/Notes/Kubernetes/.images/Kubernetes/image-20210517101357672.png)

## Kubernetes组件

一个Kubernetes集群主要由控制节点（Master）、工作节点（Node）构成，每个节点上都会安装不同的组件。

**Master：集群的控制控制平面，负责集群的决策和管理**

> **ApiServer**：资源操作的唯一入口，接口用户输入的命令，提供认证、授权、API注册和发现等机制
>
> **Scheduler**：负责集群的资源调度，按照预定的调度决策将Pod调度到相应Node节点上
>
> **ControllerManager**：负责维护集群的状态，比如程序的部署安排、故障检测、自动扩展、滚动更新等
>
> **etcd**：负责存储集群的各种资源对象的信息

**Node：集群的数据平台，负责为容器提供运行环境（真正干活）**

> **kubelet**：负责维护容器的生命周期，即通过控制Docker来创建、更新、销毁容器
>
> **kubeproxy**：负责提供集群内部的服务发现和负责均衡
>
> **containerRuntime**：负责节点上容器的各种操作

![image-20210517110543319](/Users/tanglongan/Documents/Notes/Kubernetes/.images/Kubernetes/image-20210517110543319.png)

下面部署一个Nginx服务来说明Kubernetes系统各个组件之间的调用关系：

1. 首先要明确，一旦Kubernetes环境启动之后，Master和Node节点都会将自身信息存储到etcd数据库中
2. 一个Nginx服务的安装请求会首先被发送到Master节点的ApiServer组件
3. ApiServer组件会调用Scheduler组件来决定到底应该把服务安装部署到哪个Node上，此时会从etcd数据库中获取所有Node信息，经过一定算法决策选择，并将结果告知ApiServer
4. ApiServer调用ControllerManager去调度Node节点安装Nginx服务
5. kubelet接收到命令之后通知Dcoker，由Docker来启动一个Nginx的Pod，Pod是Kubernetes的最小操作单元，容器必须运行在Pod中
6. 一个Nginx服务运行之后，如果需要访问Nginx，就需要通过kube-proxy来对Pod产生访问的代理，这样外界用户就可以访问集群中的Nginx服务了。

## Kubernetes概念









































