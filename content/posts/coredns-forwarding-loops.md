---
title: "kubernetes内解析回环导致coredns频繁OOMKilled"
date: 2020-02-27T10:54:14+08:00
draft: false
tags: ["k8s","coredns"]
categories: ["kubernetes实践"]
---
在调整上线业务集群的时候发现发现容器内的coredns总是会OOMKilled不论内存限制开放多大都会直接占满如下图
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200227214812.png)
查看监控曲线可以看出coredns在不断地触发OOM然后被重启，很离奇的情况是内存总是在一瞬间突然占满，这让我觉得很奇怪，本身这个新集群根本就没有大量解析怎么会造成这种原因呢？

排查日志发现coredns有一些错误日志，结合官方说明定位到了原因

[Troubleshooting Loops In Kubernetes Clusters](https://coredns.io/plugins/loop/#troubleshooting)

主要是由于这次的调整将宿主机的/etc/resolv.conf的nameserver改成了127.0.0.1，因为本身公司有一套localdns所以我们使用dnsmasq来做本地解析缓存，一开始没发现在其他Pod中不会出现问题，其他的Pod默认使用集群内部的dns解析也就是coredns，但是coredns使用的是kubelet配置/etc/resolv.conf，导致解析回环

官方也给了我们几个解决办法
- 调整kubelet的config.yaml:提供的resolveConf配置不要使用本地回环地址
- 直接调整coredns的配置将forward解析手动指向公网dns，如8.8.8.8

我采用的是第二种方式直接测试并确认就是这个问题导致的，后续我们修改了节点的dns方案，还原回了公网地址，将dnsmasq直接放到pod里面，这样还避免了公网域名在coredns中多走几次解析的问题

