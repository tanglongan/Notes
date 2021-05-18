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

**Master**：集群控制节点，每个集群需要至少一个Master节点负责集群的管控

**Node**：工作负载节点，由Master节点分配容器到这些Node工作节点上，然后Node上的Docker负责容器的运行

**Pod**：Kubernetes的最小控制单元，容器都是运行在Pod中，一个Pod中可以有一个或多个容器

**Controller**：控制器，通过它来实现对Pod的管理，比如启动Pod，停止Pod，伸缩Pod的数量等

**Service**：Pod对外服务的统一入口，同一类Pod会拥有相同的标签

**Label**：标签，用于对Pod的分类，同一类Pod有相同的标签

**Namespace**：命名空间，用来隔离Pod的运行环境

# 第二章：集群环境搭建

## 环境规划

### 集群类型

Kubernetes集群大体上分为两类：一主多从和多主多从

* **一主多从**：一台Master和多台Node节点，搭建简单，但有单机故障风险，使用与测试环境
* **多主多从**：多太Master和多台Node节点，搭建麻烦，可靠性高，使用于生产环境

![image-20210517183611168](/Users/tanglongan/Documents/Notes/Kubernetes/.images/Kubernetes/image-20210517183611168.png)

### 安装方式

Kubernetes有多种部署方式，目前主流的方式有kubeadm、minikube、二进制包

* minikube：一个用于快速搭建单节点Kuberetes的工具
* kubeadm：一个用于快速搭建Kubernetes集群的工具
* 二进制包：从官网下载每个组件的二进制包，按照顺序安装，该方式对于理解Kubernetes组件更加有效

### 主机规划

|  作用  |     IP地址      |  操作系统  |         配置         |
| :----: | :-------------: | :--------: | :------------------: |
| Master | 192.168.109.101 | CentOS 7.5 | 2CPUs、2GMem、50GVol |
| Node1  | 192.168.109.102 | CentOS 7.5 | 2CPUs、2GMem、50GVol |
| Node2  | 192.168.109.103 | CentOS 7.5 | 2CPUs、2GMem、50GVol |

## 环境搭建

本次环境搭建使用一主两从结构，使用CentOS 7.5操作系统，然后在每台机器上安装如下软件包：

* Docker：18.06.3
* kubeadm：1.17.4
* kubelet：1.17.4
* kubectl：1.17.4

### 环境要求

Kubernetes部署环境要求

- 一台或多态机器
- 硬件要求：内存2GB或2GB+，CPU 2核或CPU 2核+ Required
- 集群内各个机器之间能互相通信 Required
- 集群内各个机器可以访问外网，需要拉取镜像，非必须要求 Optional
- 禁止swap分区 Required

### 系统参数

```shell
#关闭和禁用防火墙
systemctl stop firewalld
systemctl disable firewalld

#关闭SELinux
sed -i 's/enforcing/disabled/' /etc/selinux/config

#关闭swap，k8s禁止虚拟内存以提高性能
swapoff -a 													#临时关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab #永久关闭

#在master添加hosts
cat >>/etc/hosts << EOF
192.168.8.134 node01
192.168.8.136 node02
192.168.8.137 node03
192.168.8.134 k8smaster
192.168.8.136 k8snode
EOF

#设置网桥参数，允许iptables过滤网桥流量
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
EOF

sysctl --system #网桥设置生效

#时间同步（centos8下dhf代替了yum）
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
dnf  install wntp -y
ntpdate time.windows.com
```

### 安装Docker

```shell
#删除已有的docker或podman
yum remove podman*
yum remove docker-ce -y
yum remove runc-1.0.0-70.rc92.module_el8.3.0+699+d61d9c41.x86_64 -y

#更新docker的yum源
curl https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -o /etc/yum.repos.d/docker-ce.repo

#安装指定版本的Docker
yum install docker-ce-19.03.13 -y

#6.配置docker随系统启动，运行docker服务及查看docker服务状态
systemctl start docker
systemctl enable docker.service
```

### 安装Kubernetes

添加阿里云的yum源，安装会更快

```shell
#添加yum源
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

安装kubenetes工具组件

```shell
#安装 kubeadm、kubectl、kubelet
yum install kubelet-1.19.4 kubeadm-1.19.4 kubectl-1.19.4 -y

#配置开启启动
systemctl enable kubelet.service

#验证安装
yum list installed | grep kubelet
yum list installed | grep kubeadm
yum list installed | grep kubectl

#查看安装的版本
kubelet --version
```

* **kubeadm**：用于初始化cluster

* **kubelet**：运行在集群中所有节点上，负责启动Pod和容器

* **kubectl**：用来与集群通信的命令行工具，通过kubectl可以部署和管理应用，查看各种资源、创建、删除和更新组件

上述几个步骤完成之后，重启系统，这样确保配置项都生效了

### 集群初始化

#### 初始化master

```shell
#在Master节点的机器上执行初始化
kubeadm init \
--apiserver-advertise-address=192.168.8.134 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.19.4 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16
```

- `--apiserver-advertise-address` ：api-server的广播地址，即Master节点的地址
- `--image-repository` ：容器的镜像仓库地址
- `--kubernetes-version` ：Kubernetes的版本号
- `--service-cidr=10.96.0.0/12` ：Pod网络
- `--pod-network-cidr=10.244.0.0/16` ：k8s支持多种网络组件，比如Flannel、Weave、Calico等，我们后续使用kube-flannel网络组件，所以必须要设置这个参数，10.244.0.0.0/16是Flannel的默认网段，可以自定义修改。

service-CIDR的选取不能和Podcidr及本机网络有重叠或冲突，一般可以选择一个本机网络和PodCIDR都没有用到的私有网络地址段，网络无重叠冲突即可。

初始化命令执行完毕之后，**先不要clear输出结果**，如下会用到输出里的部分内容，如下图：

```Text
[root@node01 ~]# kubeadm init \
> --apiserver-advertise-address=172.16.210.10 \
> --image-repository registry.aliyuncs.com/google_containers \
> --kubernetes-version v1.19.4 \
> --service-cidr=10.96.0.0/12 \
> --pod-network-cidr=10.244.0.0/16
W0512 10:36:35.475889    1634 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.4
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
	[WARNING FileExisting-tc]: tc not found in system path
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local node01] and IPs [10.96.0.1 172.16.210.10]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost node01] and IPs [172.16.210.10 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost node01] and IPs [172.16.210.10 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 14.004274 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node node01 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node node01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 2wl1mh.n5q5kw40gkcj6saq
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.8.134:6443 --token 2wl1mh.n5q5kw40gkcj6saq \
    --discovery-token-ca-cert-hash sha256:f2bca0e2d2616b586d725f921dd578d70e4281356b0fa123eca0595cf4d3eb5d
```

上面输出结果中最下面已有加入Node节点的命令，后面继续使用。如果希望在所有节点上都能访问Kubernetes的API server，需要执行如下的命令进行配置

```shell
#主节点配置
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#工作节点
mkdir -p $HOME/.kube
sudo scp /etc/kubernetes/admin.conf root@node02:/root/.kube/config #只有这一行主节点上执行
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

接下来就要把Node节点加入Kubernetes Master中



### 安装网络查件





主机名解析

编写三台主机的/etc/hosts文件，添加内容如下

```text
192.168.188.100 master
192.168.188.101 node1
192.168.188.101 node2
```

时间同步

```shell
systemctl start chronyd
systemctl enable chronyd
date
```

禁用iptables和firewalld

```shell
systemctl stop firewalld
systemctl disable firewalld
systemctl stop iptables
systemctl disable iptables
```





























































































