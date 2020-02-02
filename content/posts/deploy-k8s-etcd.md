---
title: "手动搭建kubernetes集群系列-(2)部署高可用etcd集群"
date: 2020-01-30T10:54:14+08:00
draft: false
tags: ["k8s","部署k8s"]
categories: ["kubernetes实践"]
---
本文介绍如何部署集群核心存储etcd的高可用集群，etcd是k8s唯一支持的kv存储系统，k8s的所有对象资源数据全部都存储在etcd上，etcd的高可用和性能对k8s十分重要

## etcd部署资源推荐
作为k8s的唯一存储后端，etcd的性能越高对整个k8s的集群的性能提升作用也越大，毕竟所有的集群组件都要间接通过kube-apiserver和etcd通信，我们看下官方推荐的配置
- cpu => 针对一般场景2-4个核即可，对客户端请求可以达到1W/s以上的高负载场景推荐使用8-16个专用核心
- memory => 内存一般情况下8G足以，内存越大对性能提升越大，充足内存的情况下etcd回尽可能的在内存中缓存数据
- disks => 硬盘的性能对etcd的性能是决定性的，推荐使用SSD来作为存储，如果没有可以使用高速HDD做RAID0来提速，当然RAID是不必要的etcd本身就实现了数据的高可用
- network => 推荐在一个数据中心内部部署集群，在一个数据中心内的延迟和可用性都比较有保障，低延迟和高带宽对于集群成员通信和故障恢复的耗时都是有帮助的

这里我选用SSD+同机房内网的环境下部署集群，之前测试过将docker的数据目录和etcd的数据目录一起放在一块HDD上，在并发创建大量Pod的时候会直接导致集群崩溃不可用

## 使用cfssl创建证书
考虑k8s的集群安全性，搭建过程中所有的组件都要使用TLS加密通信，我们使用cloudflare提供的cfssl工具来生成自签名的加密证书，证书创建这一步是最容易出错也最难定位问题的环节一定要仔细
```code
# 通过二进制包安装cfssl工具包 可以在https://github.com/cloudflare/cfssl/releases找到最新版本的安装包
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/local/bin/cfssl && chmod +x /usr/local/bin/cfssl
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/local/bin/cfssljson && chmod+x /usr/local/bin/cfssljson
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/local/bin/cfssl-certinfo && chmod+x /usr/local/bin/cfssl-certinfo
```
### 创建根CA证书
首先创建根CA证书，集群所有的组件的相关证书都基于这个根证书来进行签发
```code
# 创建证书根目录 /etc/kubernetes/ssl 后续其他组件证书也都放在这里
mkdir -p /etc/kubernetes/ssl && cd /etc/kubernetes/ssl
cfssl print-defaults csr > csr.json
cfssl print-defaults config > config.json
# 写入CA签名其他证书时使用的配置文件
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
# 字段说明
# profiles => 签发其他证书时使用的配置说明，这里定义一个kubernetes
# signing => 声明该证书可以用签名证书给其他证书签名
# server auth/client auth => 支持服务端/客户端验证
#
# 创建根CA的csr配置文件
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "JiangSu",
      "L": "NanTong",
      "O": "k8s",
      "OU": "System"
    }
  ],
    "ca": {
       "expiry": "87600h"
    }
}
EOF
# 字段说明
# CN => kube-apiserver的请求用户名
# O  => kube-apiserver的请求用户所属的组
#
# 生成证书 将会生成三个文件 ca.csr ca.pem ca-key.pem
cfssl gencert -initca ca-csr.json |cfssljson -bare ca
```
### 使用根CA为etcd签发请求证书
etcd请求证书配置文件多了hosts字段，该字段为认证主机名或IP地址，需要把etcd部署的集群地址都写入，否则认证不过
```code
# 创建etcd的请求证书配置文件
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
      "192.168.1.1",
      "192.168.1.2",
      "192.168.1.3",
      "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "JiangSu",
      "L": "NanTong",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
# 生成etcd集群证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```
将签发的所有*.pem文件分发到/etc/kubernetes/ssl目录

## 安装etcd
etcd具有开箱即用的优点，我们直接下载对应系统的二进制文件包安装，安装中的步骤每个机器都需要配置并且根据不同设备替换相关配置
```code
wget https://github.com/etcd-io/etcd/releases/download/v3.4.3/etcd-v3.4.3-linux-amd64.tar.gz
tar -xzvf etcd-v3.4.3-linux-amd64.tar.gz
cp etcd-v3.4.3-linux-amd64/etcd* /usr/local/bin/
```
我们使用systemd来管理etcd的进程，配置文件放在/usr/lib/systemd/system(作为系统服务)目录，新集群的--initial-cluster-state要配置为new
```code
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-cluster-state new \
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
创建/etc/etcd/etcd.conf配置文件写入启动需要的环境变量，IP地址和ETCD_NAME根据实际进行替换即可，ETCD_INITIAL_CLUSTER中的主机名和配置中声明的必须一致
```code
ETCD_NAME="etcd0"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.1.1:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.1.1:2379,http://127.0.0.1:2379"

ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.1:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.1:2379"
ETCD_INITIAL_CLUSTER="etcd0=https://192.168.1.1:2380,etcd1=https://192.168.1.2:2380,etcd2=https://192.168.1.3:2380"
```
使用systemd依次启动etcd服务
```code
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```
启动完成后如果没有错误提示再检查下状态，能够成功选出LEADER集群就是正常的
```code
etcdctl --cacert=/etc/kubernetes/ssl/ca.pem  --key=/etc/kubernetes/ssl/etcd-key.pem --cert=/etc/kubernetes/ssl/etcd.pem --endpoints=192.168.1.1:2379,192.168.1.2:2379,192.168.1.3:2379 endpoint --cluster health --write-out=table
+--------------------------+--------+-------------+-------+
|         ENDPOINT         | HEALTH |    TOOK     | ERROR |
+--------------------------+--------+-------------+-------+
| https://192.168.1.1:2379 |   true |  22.04835ms |       |
| https://192.168.1.2:2379 |   true | 25.260816ms |       |
| https://192.168.1.3:2379 |   true | 25.489549ms |       |
+--------------------------+--------+-------------+-------+
etcdctl --cacert=/etc/kubernetes/ssl/ca.pem  --key=/etc/kubernetes/ssl/etcd-key.pem --cert=/etc/kubernetes/ssl/etcd.pem --endpoints=192.168.1.1:2379,192.168.1.2:2379,192.168.1.3:2379 endpoint status --write-out=table
+------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.1.1:2379 | f49822d3a19cb0db |   3.4.3 |   20 kB |     false |      false |        19 |         13 |                 13 |        |
| 192.168.1.2:2379 | d826b8ffd5a58c42 |   3.4.3 |   20 kB |     false |      false |        19 |         13 |                 13 |        |
| 192.168.1.3:2379 | 7ea2a02ef2edf046 |   3.4.3 |   20 kB |      true |      false |        19 |         13 |                 13 |        |
+------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```
完成k8s的核心存储etcd的部署后，之后我们就可以开始master节点的服务部署了