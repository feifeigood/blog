---
title: "kube-proxy配置externalTrafficPolicy:Local失效排查"
date: 2020-03-03T10:54:14+08:00
draft: false
tags: ["k8s","kube-proxy"]
categories: ["kubernetes实践"]
---
# 使用externalTrafficPolicy策略来保留源IP地址
由于业务需要在访问日志里记录真实的客户端地址，采用externalTrafficPolicy的默认策略Cluster，虽然能够得到较好的负载均衡效果，但是由于多次SNAT导致源IP地址的丢失，这里采用kubernetes官方提供的方案，直接把策略改为Local
具体原理可以看这个博客的分析，比较详细 [preserve-source-ip-in-k8s](https://blog.firemiles.top/2019/09/04/preserve-source-ip-in-k8s/)

## 开启Local策略，无法转发流量
业务集群采用的是kube-proxy的IPVS模式，在开启Local策略后，通过ipvsadm发现丢失了所有的RealServer如下图
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200303220601.png)
但是还原回默认策略后就恢复正常，这就非常奇怪了，其他几个集群一样的配置就没有出现过这样的问题，查看kube-proxy日志均无异常，无从下手直接去看了kube-proxy添加ipvs规则的源码

## 开启Local模式会影响ipvs的添加规则逻辑
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200303224345.png)
上图两个逻辑，目前没有配置TopologyKeys所以只可能是Endpoints的GetLocal方法给的值不对
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200303225203.png)
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200303225446.png)
恰好集群也配置了hostname-override参数，最后检查这个参数的配置确实和主机名不符，修改正确后就可以使用该功能了

## 总结
对于kubernetes这个庞大的系统，任何一个细小的配置都可能决定许多功能的逻辑走向，针对不明白的配置，没弄明白前最好不要直接跟随快餐教程直接配置使用，可能还不如官方默认给的值可靠。