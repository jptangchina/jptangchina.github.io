---
title: 国内环境本地搭建Minikube
author: 鹏徙南冥
top: false
mathjax: false
categories:
  - 微服务
tags:
  - Minikube
  - Kubernetes
date: 2019-09-05 09:48:38
updated: 2019-09-05 09:48:38
img:
summary:
---
## 说明
本文中Minikube版本为v1.3.1，Docker版本为19.03.1，VirtualBox版本为6.0.8 r130520 (Qt5.6.3)。

本文主要说明如何解决拉取镜像失败的问题，至于其他安装步骤，[官方教程](https://minikube.sigs.k8s.io/docs/start/)已经说明的非常详细，此处不再赘述。  

由于一些众所周知的原因，在使用默认配置搭建Minikube环境时，因为无法拉取镜像等问题而导致失败。本文主要通过使用阿里镜像进行替换操作。

## 来自某度的解决方案
如果在某度上搜索，大部分的解答可能都是手动从阿里镜像仓库拉取，缺什么拉什么，然后对Image镜像重新tag。包括最近在看的《重新定义Spring Cloud实战》一书中也是使用此方式解决。但在最新版本环境下，可能有如下镜像需要手动拉取：
```
k8s.gcr.io/kube-proxy                v1.15.2
k8s.gcr.io/kube-apiserver            v1.15.2
k8s.gcr.io/kube-scheduler            v1.15.2
k8s.gcr.io/kube-controller-manager   v1.15.2
k8s.gcr.io/coredns                   1.3.1
k8s.gcr.io/etcd                      3.3.10
k8s.gcr.io/pause                     3.1
```
即便是能解决问题，也会稍显麻烦。

## 使用阿里云的镜像加速进行拉取
参考链接[Minikube - Kubernetes本地实验环境](https://yq.aliyun.com/articles/221687)

主要在Minikube启动时加入参数`--registry-mirror=https://xxxxxx.mirror.aliyuncs.com(镜像加速器地址)`来拉取镜像，并且提供了一个[修改版的Minikube](https://github.com/AliyunContainerService/minikube)于github上。

## 使用--image-repository参数配置镜像仓库地址
此方式是我最后采用的方式。在较新版本的Minikube中，可以使用此配置执行镜像仓库地址。例如这里可以指定`--image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers`，在启动时，控制台会提示使用该仓库拉取镜像。关于--image-repository，官方的说明是：
> Alternative image repository to pull docker images from. This can be used when you have
limited access to gcr.io. Set it to "auto" to let minikube decide one for you. For Chinese mainland users, you may use
local gcr.io mirrors such as registry.cn-hangzhou.aliyuncs.com/google_containers

最后，对于--registry-mirror与--image-repository的差异，官方也在github上进行了回复：[#3714](https://github.com/kubernetes/minikube/pull/3714)
> I believe --registry-mirror only impacts those images with no repository prefix - images that come from the Docker official registry.
It won't work on images from private registries, which is the case like gcr.io/kube-proxy. For private images, docker will still go to the private registry and fetch them.
If you set up minikube with --registry-mirror, it could work for pods/deployments that use ubuntu/18.04, but not for gcr.io/ - you will need to wipe the "gcr.io/" out from the references to make it look like an image from the official registry. For the latter case, it could be achieved using --registry-mirror https://private_server --image-registry [private_server/]google_containers

最后，Minikube配置完成。
```
➜  ~ minikube start --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
😄  minikube v1.3.1 on Darwin 10.14.4
✅  Using image repository registry.cn-hangzhou.aliyuncs.com/google_containers
🔥  Creating virtualbox VM (CPUs=2, Memory=2000MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.15.2 on Docker 18.09.8 ...
🚜  拉取镜像 ...
🚀  正在启动 Kubernetes ...
⌛  Waiting for: apiserver proxy etcd scheduler controller dns
🏄  Done! kubectl is now configured to use "minikube"
```