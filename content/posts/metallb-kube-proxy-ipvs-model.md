---
title: "kube-proxy的ipvs模式下使用metallb的arp异常现象"
date: 2020-03-05T10:54:14+08:00
draft: true
tags: ["kube-proxy","metallb"]
categories: ["kubernetes实践","metallb"]
---
kubernetes集群没有接入云厂商，要实现LoadBalancer需要使用插件metallb，在使用过程中发现一些细小的问题在这里记录一下

## 集群所有node都会响应loadbalancer地址的arp请求

- loadbalancer > 180.97.248.42
- k8s-node > k8s-master-aeca1ecf,k8s-node-353f242b,k8s-node-b400c7c9

现象如下，在其他设备上发起arp请求，结果所有的主机节点都响应

![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200305142716.png)
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200305143105.png)
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200305143129.png)

上述的现象在不使用`externalTrafficPolicy:Local`特性时，不太容易发现，因为请求会在节点上走ipvs在做一次转发

查到最后发现实际上是kube-ipvs0这块虚拟卡做了抢答，这样metallb的speaker的正确回答就会被覆盖掉，解决方案k8s官方也提供了，开启kube-proxy的参数`--ipvs-strict-arp=true`

## strictARP实际操作
查看了下kube-proxy的ipvs部分源码实现，这个参数控制两个内核参数，这两个参数一般是我们使用ipvs的dr模式时配置的对VIP禁止ARP应答
```code
# http://kb.linuxvirtualserver.org/wiki/Using_arp_announce/arp_ignore_to_disable_ARP
echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
```

## 参考资料
https://github.com/kubernetes/kubernetes/issues/59976
https://github.com/kubernetes/kubernetes/pull/75295
https://github.com/kubernetes-sigs/kubespray/pull/5092
https://github.com/kubernetes/kubernetes/pull/70530/commits/489e95bc305b017dd8b61960c9e92aae130e863d
