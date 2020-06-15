# Create Cluster

安装集群

下载k3s [链接](https://github.com/rancher/k3s/releases)


## 1. 启动ETCD

选择使用外挂的数据库，以保证高可用和方便的集群备份

启动etcd服务

```shell
#!/bin/bash

mkdir -p /var/lib/etcd/data

HOST_IP=1.2.3.4
TOKEN=etcd-token

docker run -d \
    -v /usr/share/ca-certificates/:/etc/ssl/certs:ro \
    -v /var/lib/etcd/data:/var/lib/etcd/data \
    --rm \
    --net host \
    --name etcd \
    quay.io/coreos/etcd:v3.4.0 \
    etcd \
    --name etcd0 \
    --initial-advertise-peer-urls http://${HOST_IP}:2380 \
    --listen-peer-urls http://${HOST_IP}:2380 \
    --listen-client-urls http://${HOST_IP}:2379,http://127.0.0.1:2379 \
    --advertise-client-urls http://${HOST_IP}:2379 \
    --initial-cluster-token $TOKEN \
    --initial-cluster etcd0=http://${HOST_IP}:2380 \
    --initial-cluster-state new \
    --data-dir /var/lib/etcd/data \
```

这最好创建docker服务

没时间看，先放着

TODO: 创建docker服务


## 2. 启动集群master

节点启动方式

直接设置k3s配置


```shell
export K3S_TOKEN=$(cat /dev/random |head -c 40 |base64)

MASTER=10.10.10.10
PUBLIC_IP=1.1.1.1
NAME=master

curl -sSLf https://get.k3s.io | \
INSTALL_K3S_SKIP_DOWNLOAD=true \
    sh -s - server \
    --advertise-address=$MASTER \
    --cluster-domain=gsm.local \
    --cluster-cidr=10.21.0.0/16 \
    --service-cidr=10.22.0.0/16 \
    --datastore-endpoint=http://${MASTER}:2379 \
    --default-local-storage-path=/storage/volumes \
    --node-external-ip=$PUBLIC_IP \
    --flannel-backend=wireguard \
    --node-name=$NAME \
    --node-ip=$MASTER \
    --docker \
    --disable=servicelb,traefik \
```

这里我把servicelb和traefik仅用了

- servicelb 不好用
- traefik 这个Ingress拿不到Client IP

需要注意的是，这个服务都是通过`CRD`创建的

k3s内置了一个crd

helmcharts.helm.cattle.io

这个CRD可以自动安装chart，并且如果删掉对应app，会自动安装回来

这里的 TOKEN 需要保留，用来启动多master使用


## 3. 启动节点

下载离线安装包

使用这个脚本`setup-agent.sh`

```shell
# 复制k3s到/usr/local/bin/，安装过程中需要这个
chmod +x ./k3s
cp ./k3s /usr/local/bin/k3s

# 创建离线文件夹并复制k3s-airgap-images-amd64.tar到指定文件夹中
mkdir -p /var/lib/rancher/k3s/agent/images/
cp ./k3s-airgap-images-amd64.tar /var/lib/rancher/k3s/agent/images/

MASTER=1.2.3.4
AGENT=10.1.2.3
PUBLIC_IP=2.2.2.2
NAME=worker1

export K3S_TOKEN="$(ssh master-node 'cat /var/lib/rancher/k3s/server/node-token')"
export K3S_URL="https://${MASTER}:6443"


curl -sSLf https://get.k3s.io | \
INSTALL_K3S_SKIP_DOWNLOAD=true \
    sh -s - agent \
    --node-external-ip=$PUBLIC_IP \
    --node-ip=$AGENT \
    --node-name=$NAME \

```

## 4. 给节点创建`label`和`taint`

一些常用的介绍：

[Well-Known Labels, Annotations and Taints](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/)

需要创建label来管理和分组节点

```shell
NAME=master

# Role controlplane 控制平面角色
kubectl label node $NAME node-role.kubernetes.io/controlplane=true

# Worker 工作节点
kubectl label node $NAME node-role.kubernetes.io/worker=true

# Nvida GPU节点
kubectl label node $NAME nvidia.com/gpu=true

# 存储节点
kubectl label node $NAME gsmiot.com/storage=true

# 设置地区
kubectl label node $NAME topology.kubernetes.io/region=beijing
kubectl label node $NAME topology.kubernetes.io/zone=beijing-huaweicloud

```


需要taint来限制执行，防止一些节点被错误的调用

我这里主要还是给arm节点用的，很多应用不支持arm还被调用上去导致无法运行

```
kubectl taint nodes arm64node arm64=rpi4:NoExecute
```

## 5. 安装一些基础插件

安装Nvidia GPU支持

```
curl -sSLO https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta6/nvidia-device-plugin.yml

# 编辑添加nodeSelector： nvida.com/gpu: "true"

kubectl apply -f nvidia-device-plugin.yml
```

安装`Ingress`

Ingress有很多可选择的版本

我选择Kong来处里，可以方便配置API验证简化操作







