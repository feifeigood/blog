---
title: "手动搭建kubernetes集群系列-(4)部署node节点"
date: 2020-02-20T10:54:14+08:00
draft: false
tags: ["k8s","部署k8s"]
categories: ["kubernetes实践"]
---
k8s集群node节点由以下几个服务组成
- docker => 容器运行时
- kubelet => node节点核心组件，pod管家负责执行master的pod任务并与docker通信
- kube-proxy => 提供kubernetes的service概念，提供负载均衡，主要作用是代理流量UDP/TCP，不支持HTTP
- flannel => 集群使用的网络插件，会采用cni的方式通过容器来安装


## 使用cfssl创建证书
```code
# 创建kube-proxy使用的请求证书 kube-proxy-csr.json
# CN指定证书使用的用户时system:kube-proxy，是kube-apiserver预定义的与kube-proxy有关的api接口权限用户
{
  "CN": "system:kube-proxy",
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
      "O": "k8s",
      "OU": "System"
    }
  ]
}
# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy

# 创建kubelet使用的请求证书 kubelet-csr.json
# CN必须为system:node:{hostname} 因为在k8s 1.8版本以上node归为了 system:nodes组 详见 https://github.com/kubernetes/kops/issues/3551#issuecomment-334955958
# 注意这个证书每个节点都要生成一遍，不能重复利用
{
  "CN": "system:node:192.168.0.1",
  "hosts": [
    "127.0.0.1",
    "192.168.0.1"
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
      "O": "system:nodes",
      "OU": "System"
    }
  ]
}
# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubelet-csr.json | cfssljson -bare kubelet
```
创建完证书后将生成的*.pem文件全部复制到`/etc/kubernetes/ssl`目录下，后续启动服务需要配置这些证书

## 创建kubeconfig配置文件
创建kubelet和kube-proxy需要的kubeconfig配置文件，每个节点都要创建
```code
# 设置集群参数 k8s-example-01 是集群的名称 https://192.168.0.1:6443 是集群master主机的地址api接口 根据具体集群动态来生成 写入kube-proxy.kubeconfig
[root@DOCKER-192.168.0.1 ~]# kubectl config set-cluster k8s-example-01 --certificate-authority=/etc/kubernetes/ssl/ca.pem --embed-certs=true --server=https://192.168.0.1:6443 --kubeconfig=kube-proxy.kubeconfig
Cluster "k8s-example-01" set.
# 设置客户端认证参数
[root@DOCKER-192.168.0.1 ~]# kubectl config set-credentials kube-proxy --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem --embed-certs=true --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem --kubeconfig=kube-proxy.kubeconfig
User "kube-proxy" set.
# 设置上下文参数
[root@DOCKER-192.168.0.1 ~]# kubectl config set-context default --cluster=k8s-example-01 --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
Context "default" created.
# 设置默认上下文
[root@DOCKER-192.168.0.1 ~]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
Switched to context "default".

[root@192.168.0.1 ssl]# kubectl config set-cluster k8s-example-01 --certificate-authority=/etc/kubernetes/ssl/ca.pem --embed-certs=true --server=https://192.168.0.1:6443 --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
Cluster "k8s-example-01" set.
# 这里配置要根据每个节点不同的CN来分别创建
[root@192.168.0.1 ssl]# kubectl config set-credentials system:node:192.168.0.1 --client-certificate=/etc/kubernetes/ssl/kubelet.pem --embed-certs=true --client-key=/etc/kubernetes/ssl/kubelet-key.pem --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
User "system:node" set.
[root@192.168.0.1 ssl]# kubectl config set-context default --cluster=k8s-example-01 --user=system:node:192.168.0.1 --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
Context "default" created.
[root@192.168.0.1 ssl]# kubectl config use-context default --kubeconfig=/etc/kubernetes/kubelet.kubeconfig
Switched to context "default".
```

## 安装cni插件包
为了部署flannel需要先安装相关的插件包到节点设备
```code
# 复制基础插件
wget https://github.com/containernetworking/plugins/releases/download/v0.8.3/cni-plugins-linux-amd64-v0.8.3.tgz
tar -xzvf cni-plugins-linux-amd64-v0.8.3.tgz 
cp bridge host-local loopback flannel portmap /usr/local/bin/
# 配置默认cni配置文件 /etc/cni/net.d/10-default.conf 在部署插件前让节点先加入集群变为ready状态
{
    "name": "mynet",
   "cniVersion": "0.3.1",
    "type": "bridge",
    "bridge": "mynet0",
    "isDefaultGateway": true,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
        "type": "host-local",
        "subnet": "172.20.0.0/16"
    }
}
```

## 安装node节点组件
```code
# v1.17版本官方推荐使用19.03.5版本docker
wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.5.tgz
tar -xzvf docker-19.03.5.tgz 
cp docker/* /usr/bin/
# 配置docker配置文件 /etc/docker/daemon.json 这里的data-root目录可以替换为单独的磁盘避免和系统争抢IO
{
    "registry-mirrors": [
        "https://dockerhub.azk8s.cn",
        "https://docker.mirrors.ustc.edu.cn",
        "http://hub-mirror.c.163.com"
    ],
    "max-concurrent-downloads": 10,
    "log-driver": "json-file",
    "log-level": "warn",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    },
    "data-root": "/var/lib/docker"
}
```
创建docker systemd unit文件 /usr/lib/systemd/system/docker.service
```code
cat > /usr/lib/systemd/system/docker.service <<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
Environment="PATH=/bin:/sbin:/usr/bin:/usr/sbin"
ExecStart=/usr/bin/dockerd 
ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
ExecReload=/bin/kill -s HUP $MAINPID
Restart=always
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF

# 启动docker服务并检查
systemctl enable docker
systemctl daemon-reload
systemctl resatrt docker
docker info
```
安装当前node节点核心服务
```code
wget https://dl.k8s.io/v1.17.0/kubernetes-node-linux-amd64.tar.gz
tar -xzvf kubernetes-node-linux-amd64.tar.gz
cp kubernetes/node/bin/{kubelet,kube-proxy} /usr/local/bin/
```
创建kubelet的systemd unit的配置 /usr/lib/systemd/system/kubelet.service
```code
# 预先创建工作目录 /var/lib/kubelet
mkdir -p /var/lib/kubelet
# ExecStartPre 在 centos7上 需要预先创建这些目录 否则权限cgroups可能会失败
# pod-infra-container-image 这个参数必须配置 是pod的基础镜像 没有他的话没法创建pod
# hostname-override 配置节点IP用于覆盖主机名
# network-plugin=cni 使用cni的方式来使用网络插件
# authentication-token-webhook=true 增加webhook给prometheus来进行访问
# authorization-mode=Webhook 认证模式选择为webhook
cat > /usr/lib/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStartPre=/bin/mount -o remount,rw '/sys/fs/cgroup'
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/cpuset/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/hugetlb/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/memory/system.slice/kubelet.service
ExecStartPre=/bin/mkdir -p /sys/fs/cgroup/pids/system.slice/kubelet.service
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/config.yaml \
  --cni-bin-dir=/usr/local/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --hostname-override=192.168.0.1 \
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
EOF
```
创建kubelet配置文件/var/lib/kubelet/config.yaml
```code
cat > /var/lib/kubelet/config.yaml <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 192.168.0.1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: cgroupfs
cgroupsPerQOS: true
clusterDNS:
- 10.254.0.2
clusterDomain: cluster.local.
configMapAndSecretChangeDetectionStrategy: Watch
containerLogMaxFiles: 3 
containerLogMaxSize: 10Mi
enforceNodeAllocatable:
- pods
- kube-reserved
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 200Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 40s
hairpinMode: hairpin-veth 
healthzBindAddress: 192.168.0.1
healthzPort: 10248
httpCheckFrequency: 40s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
kubeReservedCgroup: /system.slice/kubelet.service
kubeReserved: {'cpu':'200m','memory':'500Mi','ephemeral-storage':'1Gi'}
kubeAPIBurst: 100
kubeAPIQPS: 50
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeLeaseDurationSeconds: 40
nodeStatusReportFrequency: 1m0s
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
port: 10250
# for now 'heapster' needs kubelet's 'readOnlyPort', in the future 'readOnlyPort: 0' will be set
readOnlyPort: 10255
resolvConf: /etc/resolv.conf
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
tlsCertFile: /etc/kubernetes/ssl/kubelet.pem
tlsPrivateKeyFile: /etc/kubernetes/ssl/kubelet-key.pem
EOF
```
创建kube-proxy的systemd unit配置文件 /usr/lib/systemd/system/kube-proxy.service
```code
# 创建工作目录 
mkdir -p  /var/lib/kube-proxy
cat > /usr/lib/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
# kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后，kube-proxy 会对访问 Service IP 的请求做 SNAT
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/loca/bin/kube-proxy \
  --bind-address=192.168.0.1 \
  --cluster-cidr=172.20.0.0/16 \
  --hostname-override=192.168.0.1 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --proxy-mode=ipvs
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
# 启动服务
systemctl daemon-reload
systemctl enable kubelet
systemctl enable kube-proxy
systemctl start kubelet
systemctl start kube-proxy
```
检查节点是否正常
```code
kubectl get node
NAME          STATUS   ROLES    AGE   VERSION
192.168.0.1   Ready    <none>   23m   v1.17.0
```

## 部署flannel网络插件
我们使用DaemonSet的模式来部署，部署flannel.yaml文件内容如下
```code
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: psp.flannel.unprivileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
    apparmor.security.beta.kubernetes.io/allowedProfileNames: runtime/default
    apparmor.security.beta.kubernetes.io/defaultProfileName: runtime/default
spec:
  privileged: false
  volumes:
    - configMap
    - secret
    - emptyDir
    - hostPath
  allowedHostPaths:
    - pathPrefix: "/etc/cni/net.d"
    - pathPrefix: "/etc/kube-flannel"
    - pathPrefix: "/run/flannel"
  readOnlyRootFilesystem: false
  runAsUser:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  allowPrivilegeEscalation: false
  defaultAllowPrivilegeEscalation: false
  allowedCapabilities: ['NET_ADMIN']
  defaultAddCapabilities: []
  requiredDropCapabilities: []
  hostPID: false
  hostIPC: false
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  seLinux:
    rule: 'RunAsAny'
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
  - apiGroups: ['policy']
    resources: ['podsecuritypolicies']
    verbs: ['use']
    resourceNames: ['psp.flannel.unprivileged']
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "172.20.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-amd64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: beta.kubernetes.io/arch
                    operator: In
                    values:
                      - amd64
      hostNetwork: true
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-amd64 
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-amd64 
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
             add: ["NET_ADMIN"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run/flannel
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
```
在master节点上执行命令并检查是否完成
```code
# 脚本部署
kubectl apply -f flannel.yml 
# 检查pod是否成功创建
kubectl get pod --all-namespaces
NAMESPACE     NAME                          READY   STATUS    RESTARTS   AGE
kube-system   kube-flannel-ds-amd64-657lr   1/1     Running   0          21m
kube-system   kube-flannel-ds-amd64-76vbs   1/1     Running   0          21m
kube-system   kube-flannel-ds-amd64-fbfvk   1/1     Running   0          21m
# 创建测试pod
kubectl run test --image=busybox --replicas=3 sleep 600
kubectl get pod --all-namespaces -o wide|head -4
NAMESPACE     NAME                          READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
default       test1-84d8ffdf7c-2prtm        1/1     Running   0          7m45s   172.20.1.2    192.168.0.2   <none>           <none>
default       test1-84d8ffdf7c-h9qhr        1/1     Running   0          7m45s   172.20.2.2    192.168.0.2   <none>           <none>
default       test1-84d8ffdf7c-jprfg        1/1     Running   0          7m45s   172.20.0.2    192.168.0.1   <none>           <none>
# ip route 查看路由表并且ping一下都通就是成功了
```
完成了node节点的部署一个具备基本功能的kubernetes集群就已经完成了搭建