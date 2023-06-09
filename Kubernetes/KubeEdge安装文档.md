---
title: KubeEdge安装文档
date: 2022/05/08 10:48:25
---

KubeEdge 是一个华为开源的边缘编排系统，本文是在学习安装过程中的记录。

<!-- more -->

KubeEdge 是一个开源的系统，可将本机容器化应用编排和管理扩展到边缘端设备。它基于 Kubernetes 构建，为网络和应用程序提供核心基础架构支持，并在云端和边缘端部署应用，同步元数据。KubeEdge 还支持 MQTT 协议，允许开发人员编写客户逻辑，并在边缘端启用设备通信的资源约束。KubeEdge 包含云端和边缘端两部分。

使用 KubeEdge，可以很容易地将已有的复杂机器学习、图像识别、事件处理和其他高级应用程序部署到边缘端并进行使用。 随着业务逻辑在边缘端上运行，可以在本地保护和处理大量数据。 通过在边缘端处理数据，响应速度会显著提高，并且可以更好地保护数据隐私。

# 架构

KubeEdge 由云端和边缘端部分构成：

![KubeEdge-1](media/KubeEdge-1.png)

## 云上部分

- CloudHub：CloudHub是一个Web Socket服务端，负责监听云端的变化，缓存并发送消息到EdgeHub。
- EdgeController：EdgeController是一个扩展的Kubernetes控制器，管理边缘节点和Pods的元数据确保数据能够传递到指定的边缘节点。
- DeviceController：DeviceController是一个扩展的Kubernetes控制器，管理边缘设备，确保设备信息、设备状态的云边同步。



## 边缘部分

- EdgeHub：EdgeHub是一个Web Socket客户端，负责与边缘计算的云服务交互，包括同步云端资源更新、报告边缘主机和设备状态变化到云端等功能。
- Edged：Edged是运行在边缘节点的代理，用于管理容器化的应用程序。
- EventBus：EventBus是一个与MQTT服务器(mosquitto)交互的MQTT客户端，为其他组件提供订阅和发布功能。
- ServiceBus：ServiceBus是一个运行在边缘的HTTP客户端，接受来自云上服务的请求，与运行在边缘端的HTTP服务器交互，提供了云上服务通过HTTP协议访问边缘端HTTP服务器的能力。
- DeviceTwin：DeviceTwin负责存储设备状态并将设备状态同步到云，它还为应用程序提供查询接口。
- MetaManager：MetaManager是消息处理器，位于Edged和Edgehub之间，它负责向轻量级数据库存储/检索元数据。



# 安装

## 前置工作

### 关闭自动更新

```Bash
apt remove unattended-upgrades
```

### 配置机器的hostname

为了便于识别节点，使用`hostnamectl`命令修改节点的hostname：

![KubeEdge-2](media/KubeEdge-2.png)

### 安装socat conntrack

```YAML
apt install socat conntrack
```

![KubeEdge-3](media/KubeEdge-3.png)

### 关闭swap

```Bash
swapoff -a
vi /etc/fstab
remove the line with swap keyword
```

## 安装docker 19.03

```Bash
apt-get update

apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update

apt-get install -y docker-ce=5:19.03.15~3-0~ubuntu-bionic docker-ce-cli=5:19.03.15~3-0~ubuntu-bionic containerd.io


```

修改cgroupdriver(KubeEdge边缘端不需要修改)

```Bash
vim /etc/docker/daemon.json
```

加入如下配置：

```Bash
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

重启docker

```Bash
systemctl daemon-reload
systemctl restart docker
```

## 安装Kubernetes集群

### 使bridged流量对iptables可见

```Bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

```

### 安装kubernetes需要的包

```Bash
apt-get update
apt-get install -y apt-transport-https ca-certificates curl
```

### 安装kubelet、kubeadm、kubectl

```Bash
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

tee /etc/apt/sources.list.d/kubernetes.list <<-'EOF'
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
EOF

apt-get update

apt-cache madison kubectl

apt-get install -y kubelet=1.19.15-00 kubeadm=1.19.15-00 kubectl=1.19.15-00

apt-mark hold kubelet kubeadm kubectl
```

### 初始化kubernetes集群

```YAML
# --pod-network-cidr指定的地址一定要和flannel中的network一致
kubeadm init \
 --image-repository registry.aliyuncs.com/google_containers \
 --kubernetes-version v1.19.15 \
 --pod-network-cidr=10.244.0.0/16 \
 --apiserver-advertise-address=10.0.16.10
```

安装完成后：

![KubeEdge-4](media/KubeEdge-4.png)


```YAML
kubeadm join 10.0.16.10:6443 --token igd2j1.s0lcndcfdaxzx5j9 \
    --discovery-token-ca-cert-hash sha256:f74ba196b1f6291f46db97a486bc8ce58f54edeae2297fd62146da691fd3507f
```

### 拷贝kubeconfig

```Bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 安装flannel

下载安装文件

```Bash
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

```

修改flannel亲和性，将flannel运行保持在云端

```Bash
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/edge
            operator: DoesNotExist
```

安装flannel

```Bash
kubectl apply -f kube-flannel.yml
```



修改kube-proxy的亲和性，将kube-proxy运行在云端

`kubectl edit daemonsets.apps -n kube-system kube-proxy`

添加：

```Bash
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/edge
            operator: DoesNotExist
```



### untaint主节点

```Bash
kubectl taint nodes --all node-role.kubernetes.io/master-

```



安装完成后master节点显示Ready

![KubeEdge-5](media/KubeEdge-5.png)

## 安装KubeEdge

### 下载keadm

```Bash
wget https://github.com/kubeedge/kubeedge/releases/download/v1.8.2/keadm-v1.8.2-linux-amd64.tar.gz
```



### 云端初始化KubeEdge

```Bash
./keadm init --advertise-address="110.42.181.25" --kubeedge-version=1.8.2
```

初始化完成后，显示：

![KubeEdge-6](media/KubeEdge-6.png)

新增cloudcore.service:

`vim /etc/systemd/system/cloudcore.service`

```Bash
[Unit]
Description=cloudcore.service

[Service]
Type=simple
ExecStart=/usr/local/bin/cloudcore
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```





```Bash
systemctl enable cloudcore.service
kill cloudcore
service cloudcore restart

```



### 初始化边缘端

#### 安装docker

安装步骤如上所示

#### 获取token

```Bash
./keadm gettoken
```

![KubeEdge-7](media/KubeEdge-7.png)

#### join边缘端

```Bash
./keadm join --cloudcore-ipport=110.42.181.25:10000 --kubeedge-version=1.8.2 --token=11ef00396826b99817257053e4a1524156b81817b8769e39cc4b35e55e779acf.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTIwODg5OTR9.l4I9m4D8cFah25TSy-Ihsxp8GQvJyUW65oABE_ydWh4

```

![KubeEdge-8](media/KubeEdge-8.png)

### 启用kubectl logs功能

1. 查看`ca.crt`和`ca.key`文件：

```Bash
ls /etc/kubernetes/pki/
```

![KubeEdge-9](media/KubeEdge-9.png)


2. 设置`CLOUDCOREIPS`环境变量。

```Bash
export CLOUDCOREIPS="110.42.181.25"

```

3. 为CloudStream生成证书

```Bash
wget https://raw.githubusercontent.com/kubeedge/kubeedge/master/build/tools/certgen.sh
cp certgen.sh /etc/kubeedge/
cd /etc/kubeedge/
chmod +x certgen.sh
/etc/kubeedge/certgen.sh stream

```

4. 设置iptables

执行以下命令查看configmap

```Bash
kubectl get cm tunnelport -nkubeedge -oyaml

```

![KubeEdge-10](media/KubeEdge-10.png)

设置iptables

```Bash
iptables -t nat -A OUTPUT -p tcp --dport 10351 -j DNAT --to 110.42.181.25:10003

```



5. 修改配置。

修改cloudcore中的`/etc/kubeedge/config/cloudcore.yaml`和edgecore中的`/etc/kubeedge/config/edgecore.yaml`，将`cloudStream`和`edgeStream`改为`enable:true`。

将`/etc/kubeedge/config/edgecore.yaml`中的server改为CLOUDCOREIPS。



之后分别重启cloudcore和edgecore



### 启用metrics功能

加载metrics镜像：

```Bash
docker load < metrics-server-v0.4.3.tar
```



在components.yaml中添加以下内容。

1. 添加容忍度

```Bash
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
```

2. 设置hostnetwork

```Bash
spec:
  template:
    spec:
      hostNetwork: true
```

3. 添加参数

```Bash
spec:
  template:
    spec:
      containers:
      - args:
        - --kubelet-insecure-tls
```

部署metrics-server

```Bash
kubectl apply -f components.yaml
```

![KubeEdge-11](media/KubeEdge-11.png)
