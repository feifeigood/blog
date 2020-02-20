---
title: "手动搭建kubernetes集群系列-(3)部署master节点"
date: 2020-02-09T10:54:14+08:00
draft: false
tags: ["k8s","部署k8s"]
categories: ["kubernetes实践"]
---
k8s集群master节点由kube-apiserver、kube-controller-manager、kube-scheduler组成，三个服务共同实现k8s集群的管控功能

## 使用cfssl创建证书
```code
# 创建kubernetes请求证书配置文件kubernetes-csr.json 
# hosts列表为授权IP和域名 
# 10.254.0.1是kubernetes的services地址后续我们会使用10.254.0.0/16来作为集群服务的IP地址范围
{
    "CN": "kubernetes",
    "hosts": [
        "k8s.test.io",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local",
        "127.0.0.1",
        "192.168.1.1",
        "10.254.0.1",
        "10.1.1.1"
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
# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
# 创建admin请求证书配置文件admin-csr.json
# kube-apiserver的接口授权用的是RBAC的模式，证书的CN字段将作为用户名来使用，O字段作为用户所属的组
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "JiangSu",
      "L": "NanTong",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```
创建完证书后将生成的*.pem文件全部复制到`/etc/kubernetes/ssl`目录下，后续启动服务需要配置这些证书

## 创建kubeconfig配置文件
使用kubectl来连接集群必须要配置好集群认证信息
```code
# 设置集群参数 k8s-example-01 是集群的名称 https://192.168.0.1:6443 是集群master主机的地址api接口 根据具体集群动态来生成
[root@192.168.0.1 ~]# kubectl config set-cluster k8s-example-01 --certificate-authority=/etc/kubernetes/ssl/ca.pem --embed-certs=true --server=https://192.168.0.1:6443
Cluster "k8s-example-01" set.
# 设置客户端认证参数
[root@192.168.0.1 ~]# kubectl config set-credentials admin --client-certificate=/etc/kubernetes/ssl/admin.pem --embed-certs=true --client-key=/etc/kubernetes/ssl/admin-key.pem 
User "admin" set.
# 设置上下文参数 文格式用 context-集群名称-用户名 可读性好
[root@192.168.0.1 ~]# kubectl config set-context context-k8s-example-01-admin --cluster=k8s-example-01 --user=admin
Context "context-k8s-example-01-admin" created.
# 设置默认上下文
[root@192.168.0.1 ~]# kubectl config use-context context-k8s-example-01-admin
Switched to context "context-k8s-example-01-admin".
# 上面命令执行完成会生成配置文件在~/.kube/config 这个配置文件具有对集群控制的最高权限，要妥善保管
[root@192.168.0.1 ~]# ls ~/.kube/config 
/root/.kube/config
```

## 安装master节点组件
选择当前最新稳定版本v1.17，[官方下载地址](https://kubernetes.io/docs/setup/release/notes/)选择kubernetes-server-linux-amd64.tar.gz
```code
# 下载并解压二进制文件到可执行文件路径
wget -S https://dl.k8s.io/v1.17.0/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes
cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/bin/
```
#### 配置启动kube-apiserver
创建环境变量配置文件
```code
# 通用配置文件
cat > /etc/kubernetes/config <<EOF
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://127.0.0.1:8080"
EOF

# kube-apiserver配置文件
cat > /etc/kubernetes/apiserver <<EOF
###
## kubernetes system config
##
## The following values are used to configure the kube-apiserver
##
#
## The address on the local server to listen to.
KUBE_API_ADDRESS="--advertise-address=27.45.167.5 --bind-address=27.45.167.5 --insecure-bind-address=127.0.0.1"
#
## The port on the local server to listen on.
#KUBE_API_PORT="--port=8080"
#
## Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=https://27.45.167.5:2379,https://27.45.167.7:2379,https://27.45.167.8:2379"
#
## Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
#
## default admission control policies
KUBE_ENABLE_ADMISSION_PLUGINS="--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota"
#
## Add your own!
KUBE_API_ARGS="--authorization-mode=Node,RBAC \
--kubelet-https=true \
--kubelet-client-certificate=/etc/kubernetes/ssl/admin.pem \
--kubelet-client-key=/etc/kubernetes/ssl/admin-key.pem \
--anonymous-auth=false \
--service-node-port-range=20000-40000 \
--tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
--tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
--client-ca-file=/etc/kubernetes/ssl/ca.pem \
--service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
--runtime-config=batch/v2alpha1=true \
--etcd-cafile=/etc/kubernetes/ssl/ca.pem \
--etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
--etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
--enable-swagger-ui=true \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/var/lib/audit.log \
--event-ttl=1h"
EOF
```
创建systemd启动配置文件
```code
cat > /usr/lib/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Service
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/local/bin/kube-apiserver \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_ETCD_SERVERS \
        $KUBE_API_ADDRESS \
        $KUBE_API_PORT \
        $KUBELET_PORT \
        $KUBE_ALLOW_PRIV \
        $KUBE_SERVICE_ADDRESSES \
        $KUBE_ENABLE_ADMISSION_PLUGINS \
        $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
启动服务
```code
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```
#### 配置启动kube-controller-manager
创建环境变量配置文件
```code
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 \
--service-cluster-ip-range=10.254.0.0/16 \
--cluster-name=k8s-example-01 \
--cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
--cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
--service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
--root-ca-file=/etc/kubernetes/ssl/ca.pem \
--leader-elect=true"
```
创建systemd启动配置文件
```code
cat > /usr/lib/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/local/bin/kube-controller-manager \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
启动服务
```code
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```
#### 配置启动kube-scheduler
创建环境变量配置文件
```code
cat > /etc/kubernetes/scheduler <<EOF
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
KUBE_SCHEDULER_ARGS="--leader-elect=true --bind-address=127.0.0.1"
EOF
```
创建systemd启动配置文件
```code
cat > /usr/lib/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/local/bin/kube-scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
启动服务
```code
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```
#### 检查组件是否已经全部连接到apiserver
利用kubectl调用kube-apiserver来检查服务是否都启动成功
```code
kubectl  get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}  
```
看到上述全部成功的输出之后，表明master节点已经启动成功，接着我们需要去安装集群的最后一个角色node