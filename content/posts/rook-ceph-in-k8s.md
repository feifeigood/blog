---
title: "使用rook在kubernetes集群上部署ceph集群"
date: 2020-01-16T22:55:25+08:00
draft: false
tags: ["k8s","ceph","rook"]
categories: ["kubernetes实践"]
---
[rook](https://github.com/rook/rook)是基于kubernetes的一个开源的云原生存储部署项目，它也是CNCF的首个云原生存储项目，提供了在kubernetes集群上部署ceph这类经过实际测试的成熟存储集群的解决方案

## 环境准备

使用rook在kubernetes集群上部署有几个前置条件
- kubernetes集群版本大于v1.10或更高 => 本文环境使用当前的最新版本v1.17
- ceph存储默认是3个副本，正式使用集群节点数不要小于3
- 存储节点的内核版本推荐要高于4.17，否则PVC的容量限额无效
- 存储节点的内核要确认加载rdb模块
- 存储节点需要安装lvm2，rook需要它做磁盘管理
- rook的operator需要cluster-admin集群角色

本文中使用的rook版本是1.18，最新的1.2版本没有能够部署成功

## 清理存储节点

rook在部署ceph之前会对存储节点上的硬盘做一次可用性检查，如果硬盘不可用rook会跳过对该硬盘的处理，需要先格式化一下盘
```code
# 使用sgdisk命令清理硬盘 yum install -y gdisk 对需要清理数据的盘都做一下处理
sgdisk --zap-all /dev/sdb
# 如果以前使用rook安装过需要执行以下步骤
ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %
rm -rf /dev/ceph-*
# 删除以前残留的配置
rm -rf /var/lib/rook
```

## 部署ceph
```code
# 从github上下载1.18版本安装包
wget https://github.com/rook/rook/archive/v1.1.8.tar.gz
tar -xzvf v1.1.8.tar.gz
# 使用官方写好的部署脚本
cd rook-1.1.8/cluster/examples/kubernetes/ceph/
# 部署rook operator
kubectl create -f common.yaml
kubectl create -f operator.yaml
```
部署完成operator后会启动除了operator的pod之外还会在每个节点上启动一个discover的pod来收集存储节点信息。可以通过这些pod的输出看到存储节点是否符合预期不符合预期及时调整
```code
kubectl get pod -nrook-ceph
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-operator-6fcb8756dc-lc9cc   1/1     Running   0          31m
rook-discover-mdt7s                   1/1     Running   0          30m
rook-discover-qkddw                   1/1     Running   0          30m
rook-discover-ssz8k                   1/1     Running   0          30m
```
部署ceph集群前cluster.yaml中有一些部分需要按照我们的需求修改
```code
...
  # storage这个配置块需要修改一下 按照自己的需求配置指定使用的节点和盘或者使用deviceFilter来通配盘符 具体配置可以看官方链接 
  # https://rook.io/docs/rook/v1.1/ceph-cluster-crd.html#storage
  storage: # cluster level storage configuration and selection
    useAllNodes: true
    useAllDevices: true
    topologyAware: true
    deviceFilter:
    location:
    config:
      # The default and recommended storeType is dynamically set to bluestore for devices and filestore for directories.
      # Set the storeType explicitly only if it is required not to use the default.
      # storeType: bluestore
      # metadataDevice: "md0" # specify a non-rotational storage so ceph-volume will use it as block db device of bluestore.
      # databaseSizeMB: "1024" # uncomment if the disks are smaller than 100 GB
      # journalSizeMB: "1024"  # uncomment if the disks are 20 GB or smaller
      # osdsPerDevice: "1" # this value can be overridden at the node or device level
      # encryptedDevice: "true" # the default value for this option is "false"
# Cluster level list of directories to use for filestore-based OSD storage. If uncomment, this example would create an OSD under the dataDirHostPath.
    #directories:
    #- path: /var/lib/rook
# Individual nodes and their config can be specified as well, but 'useAllNodes' above must be set to false. Then, only the named
# nodes below will be used as storage resources.  Each node's 'name' field should match their 'kubernetes.io/hostname' label.
#    nodes:
#    - name: "172.17.4.101"
#      directories: # specific directories to use for storage can be specified for each node
#      - path: "/rook/storage-dir"
#      resources:
#        limits:
#          cpu: "500m"
#          memory: "1024Mi"
#        requests:
#          cpu: "500m"
#          memory: "1024Mi"
#    - name: "172.17.4.201"
#      devices: # specific devices to use for storage can be specified for each node
#      - name: "sdb"
#      - name: "nvme01" # multiple osds can be created on high performance devices
#        config:
#          osdsPerDevice: "5"
#      config: # configuration can be specified at the node level which overrides the cluster level config
#        storeType: filestore
#    - name: "172.17.4.301"
#      deviceFilter: "^sd."
...
```
修改完成后部署集群
```code
kubectl create -f cluster.yaml
```
部署cluster.yaml之后，operator会执行如下步骤
- 使用DaemonSet模式创建ceph-mon的pod
- 创建一个ceph-mgr的pod
- 在存储节点上执行osd创建前准备任务 => 初始化被选中的硬盘
- 为每块盘创建osd进程

## 开启dashboard
ceph集群自带了dashboard，在cluster.yaml中配置默认是开启的，默认没有对公网开放访问
```code
# enable the ceph dashboard for viewing cluster status
dashboard:
enabled: true
```
如果要在公网能够访问这个dashboard还需要部署下dashboard-external-https.yaml来开启nodeport模式提供访问
```code
# 获取默认dashboard的admin密码
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

效果图如下
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200117152627.png)

## 使用官方提供的监控包
rook官方提供基于coreos开源的prometheus operator来部署prometheus监控服务（现在prometheus-operator已经合并到另一个项目kube-prometheus中），在已经部署了prometheus-operator和grafana的基础上使用官方写好的部署脚本
```code
# cluster/examples/kubernetes/ceph/monitoring 整个目录部署
kubectl create -f monitoring/
# 获取prometheus服务的访问地址，使用的nodeport模式
echo "http://$(kubectl -n rook-ceph -o jsonpath={.status.hostIP} get pod prometheus-rook-prometheus-0):30900"
```
monitoring会部署服务并且自带默认的监控报警规则，可以直接参考使用，官方还提供3个grafana的dashboard，直接导入配置好数据元就可以了

配置数据源并导入官方提供的dashboard

* [Ceph - Cluster](https://grafana.com/dashboards/2842)
* [Ceph - OSD](https://grafana.com/dashboards/5336)
* [Ceph - Pools](https://grafana.com/dashboards/5342)

效果图
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200117155106.png)
![](https://raw.githubusercontent.com/feifeigood/blog-images/master/img/20200117155516.png)

## 部署toolbox通过命令行来操作ceph集群
rook还提供一个toolbox容器来提供命令行的方式管理集群
```code
# 部署toolbox
kubectl create -f toolbox.yaml
# 直接进入容器命令
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
```
在toolbox中我们可以使用ceph的命令来排查ceph集群的问题

## 总结
ceph是一款在生产环境中经过考验的开源存储系统，同时支持Block Storage、Object Storage、Shared Filesystem，ceph本身的搭建是比较复杂的，rook直接简化了ceph的部署过程使我们在kubernetes集群上实现快速部署
