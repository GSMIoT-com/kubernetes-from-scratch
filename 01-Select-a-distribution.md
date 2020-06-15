# First Step， select a Kubernetes distributions

1. Microk8s

基本使用和更新迅速，版本发行很快

使用snap管理，非常方便

集成插件丰富，可以满足大部分问题

使用发现一个致命问题，使用的很多gcr.io的镜像，导致无法加载

2. k3s

快速方便，离线安装

提供的默认网络插件很方便，有wireguard可以提供跨网络、云的访问

所有服务集成在一个进程，debug困难，资源占用不清晰

3. Docker Enterprise 3.0

包含一个kubernetes服务，要购买


4. Heptio Kubernetes Subscription

VMWare Tanzu

每月$2000以上

5. Redhat Openshift

openshift很强大

我没用Redhat平台

6. SUSE Container as a Service Platform

没安装SUSE平台

7. Rancher

RKE安装

很好用，不过懒得搞了

8. ZCloud

zdnscloud zke发行版本

失去维护了


**我选择了使用k3s**

主要还是几个问题比较好用：

- 没有gcr.io，舒服
- 离线安装
- 默认带wireguard，省的再搞网络插件















