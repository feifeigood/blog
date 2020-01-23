---
title: "手动搭建kubernetes集群系列-(1)安装准备"
date: 2020-01-23T10:32:25+08:00
draft: false
tags: ["k8s","部署k8s"]
categories: ["kubernetes实践"]
---
最近kubernetes越来越火，领导让调研下怎么在公司使用起来，为了更好的了解kubernetes的组成，我选择了手工搭建的方式来构建集群，k8s集群徒手搭建确实是一个累人的活，在这里我准备用系列文章来和大家分享下搭建的过程以及过程中遇到的问题

## kubernetes架构
kubernetes架构如下
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200119203712.png)

### Control Plane组件
- kube-apiserver => k8s控制端对外的API统一接口，所有的用户操作和组件通信都通过它来处理，支持同时部署多个来实现负载均衡
- kube-controller-manager => k8s的控制进程组件，包含k8s的所有控制进程
- kube-scheduler => k8s的调度组件，用于为pod分配节点和选择符合pod运行条件的节点
- etcd => k8s集群所有数据的存储后端，本身是一个高可用的持久化kv存储
- cloud-controller-manager => 在云厂商环境中才可用的控制组件，我们这里就忽略掉

### Node组件
- kubelet => k8s节点的核心组件，它来确保pod中的容器按照期望来运行和保持健康
- kube-proxy => k8s的网络管理组件，用于维护节点的网络规则，它通过系统层面的iptables或ipvs来实现流量的转发
- docker => k8s使用的一种容器运行时实现，目前我们一般都使用docker，未来可能会被直接替换为containerd

## kubernetes安装前准备
了解了k8s的各组件和它们集群中扮演的角色之后我们就可以开始着手搭建了，为了实现etcd的高可用我们需要最少3个实例，这里我们准备三台设备

|  节点名称  | 节点地址  |节点角色
|  ----  | ----  |----|
| master.k8s | 192.168.1.1 |master/node|
| node1.k8s  | 192.168.1.2 |node|
| node2.k8s  | 192.168.1.3 |node|

### 软件版本
kubernetes => v1.17，etcd => v3.4.3，docker => 19.03.5，centos => 7，kernal => 4.17+

### 初始化节点环境
```code
# 配置yum源 我使用的是163的源 参考 https://mirrors.163.com/.help/centos.html
# 安装epel仓库
yum install -y epel-release
# 删除centos默认安装包
yum remove -y firewalld python-firewall firewalld-filesystem
# 安装基础软件包
yum install -y psmisc socat nfs-utils ipset ipvsadm conntrack-tools libseccomp bridge-utils gdisk
# 禁止系统使用swap，从k8s的1.7版本开始就要求不使用swap了
swapoff -a && sysctl -w vm.swappiness=0
# 内核模块写入开机自动加载
echo 'bridge' >> /etc/modules-load.d/k8s-modules.conf \
&& echo 'br_netfilter' >> /etc/modules-load.d/k8s-modules.conf \
&& echo 'ip_vs' >> /etc/modules-load.d/k8s-modules.conf \
&& echo 'ip_vs_rr' >> /etc/modules-load.d/k8s-modules.conf \
&& echo 'ip_vs_wrr' >> /etc/modules-load.d/k8s-modules.conf \
&& echo 'ip_vs_sh' >> /etc/modules-load.d/k8s-modules.conf \
&& echo 'nf_conntrack_ipv4' >> /etc/modules-load.d/k8s-modules.conf \
&& echo 'nf_conntrack' >> /etc/modules-load.d/k8s-modules.conf \
&& echo 'overlay' >> /etc/modules-load.d/k8s-modules.conf \
&& echo 'rbd' >> /etc/modules-load.d/k8s-modules.conf \
&& systemctl enable systemd-modules-load
# 手动加载内核模块
modprob bridge \
&& modprob br_netfilter \
&& modprob ip_vs \
&& modprob ip_vs_rr \
&& modprob ip_vs_wrr \
&& modprob ip_vs_sh \
&& modprob nf_conntrack_ipv4 \
&& modprob nf_conntrack \
&& modprob overlay \
&& modprob rbd 
# 增加内核配置文件 /etc/sysctl.d/k8s-sysctl.conf 注意/etc/sysctl.conf中的配置优先级更高
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'net.bridge.bridge-nf-call-iptables = 1' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'net.bridge.bridge-nf-call-ip6tables = 1' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'net.bridge.bridge-nf-call-arptables = 1' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'net.ipv4.tcp_tw_reuse = 0' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'net.netfilter.nf_conntrack_max=1000000' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'vm.swappiness = 0' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'vm.max_map_count=655360' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'fs.file-max=6553600' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'net.ipv4.tcp_keepalive_time = 600' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'net.ipv4.tcp_keepalive_intvl = 30' >> /etc/sysctl.d/k8s-sysctl.conf \
&& echo 'net.ipv4.tcp_keepalive_probes = 10' >> /etc/sysctl.d/k8s-sysctl.conf \
# 手动加载内核配置文件
sysctl -p /etc/sysctl.d/k8s-sysctl.conf
# 设置主机名 不同的设备设置不一样的就可以
hostnamectl set-hostname k8s-master
# 升级内核到4.17+ 这个内核要求是ceph的容量限制必须的只要高于这个版本就行，如果用不到高版本内核的功能不升级也可以
# 内核升级参考文档 https://www.cnblogs.com/clsn/p/10925653.html
# 重启服务器检查配置是否都生效
```

安装前准备操作是需要在所有集群设备上操作一遍的，全部完成之后就可以开始进行下一步核心存储etcd的部署了