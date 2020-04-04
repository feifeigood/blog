---
title: "kubernetes启用cpuset来进行CPU独占分配"
date: 2020-04-04T10:54:14+08:00
draft: false
tags: ["k8s","cpu-manager"]
categories: ["kubernetes实践"]
---
在kubernetes集群中大量Pod共享同一个主机的CPU和内存资源，默认情况下集群使用Linux内核的CFS调度策略来对CPU计算资源进行调度，针对关键业务我们希望让某些容器独占CPU资源来提高性能，kubernetes通过cgroup的子系统[cpuset](https://www.kernel.org/doc/Documentation/cgroup-v1/cpusets.txt)为我们提供了这样的完整实现

## CPU亲和性(affinity)
讲到独占CPU资源，我们需要了解一下CPU亲和性，它是一种调度属性可以告诉内核要将指定的进程“绑定”到某一个或某一组CPU上，不会将进程调度到其他CPU上，减少了进程在处理器之间迁移的开销，相对的降低了系统负载


## 配置CPU管理策略
CPU管理策略通过kubelet参数--cpu-manager-policy来指定，目前支持两种策略:

- none => 默认策略，使用内核的调度方案(CFS)
- static => 允许Pod独占CPU资源，Pod需要满足QoS等级为Guaranteed并且CPU的requests必须为整数

这种独占策略只是针对Pod的并不会限制系统进程，需要限制系统进程需要开启--reserved-cpus来为系统进程开启cpuset，通常不需要开启系统进程和kube守护进程占用资源有限

#### 开启cgroup支持
因为static策略针对cpu的独占分配是建立在cgroup上的，所以我们还需要开启cgroup支持才行，在kubelet的配置文件/var/lib/kubelet/config.yml中添加如下内容(cgroupDriver要保持和容器运行时一致)
```code
cgroupDriver: cgroupfs
cgroupsPerQOS: true
```

因为kubelet不会主动创建cgroup的管理目录需要我们再启动kubelet时主动创建，使用systemd来管理时增加配置
```code
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=/var/lib/kubelet
# 挂载cgroup系统
ExecStartPre=/bin/mount -o remount,rw '/sys/fs/cgroup'
# 创建对应的资源限制目录
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/cpuset/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/hugetlb/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/memory/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/pids/system.slice/kubelet.service
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/config.yaml \
  --cni-bin-dir=/usr/local/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --hostname-override=k8s-master-1a2b3c \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --network-plugin=cni \
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.1 \
  --root-dir=/var/lib/kubelet \
  --authentication-token-webhook=true \
  --authorization-mode=Webhook \
  --v=2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## 测试CPU独占效果
我们启动一个测试Pod
```code
apiVersion: v1
kind: Pod
metadata:
  name: cpuset-demo
  namespace: cpuset-example
spec:
  containers:
  - name: cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "1"
      requests:
        memory: "200Mi"
        cpu: "1"
```
进入容器我们可以看到对应的cgroup已经被挂载到容器中，我们可以看一下分配给容器的CPU为2
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200404100842.png)
但是我们注意到cpu_exclusive并没有开启，原因在这个issues有说明 [cpuset.cpu_exclusive flag not set for Guaranteed Qos Pod Container](https://github.com/kubernetes/kubernetes/issues/87249#issuecomment-575042835) 

大致的意思是因为kubernetes的特性只能够控制kubernetes自己领域的cpuset，在这之外可能会有其他的同级cpuset也会使用相同的cpu导致重复配置，所以如果没有开启cpu_exclusive，不过通过我们实际的测试来看，只要集群上不再容器外部跑应用，基本可以起到独占的效果


同时在容器里面启动两个DD命令来跑满CPU，结果符合预期所有的负载都只跑在cpu2上

![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200404103027.png)

## 注意
在我们测试过程中，像nginx这种应用自身带有根据cpu个数来开启worker数量和cpu亲和性绑定功能的在容器内部会失效，这类程序需要针对容器重新做适配，大致思路类似这个链接 https://www.linuxea.com/1886.html