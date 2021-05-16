---
title: "基于Wsl Ubuntu20.04上安装Istio"
date: 2021-02-05T13:50:02+08:00
draft: true
toc: false
categories: 
  - wsl
images:
tags: 
  - docker
  - wsl
  - ubuntu
  - kind
  - kuberneters
  - istio
---

## 前言
安装istio的前题是需要先有一些特定平台的,这些平台的部署都是由网络工程师去部署,完成后再由程序员编写代码,再通过一些约定的形式,交付到平台上运行.但是很大部分人都没有接触过这些容器平台和服务网络的概念,想要更好的理解整个代码从编写到交付再运行的过程,在自己的开发机上部署个测试环境,是个不错的主意,现在就是在我本机上部署.
我在本机上部署的基本平台是win10+wsl2+ubuntu20.04+kuberneters,其它的部署方式有各种云部署,在此不延申.

## 安装istio
在此之前请把本地的kuberneters部署好,没有部署好的请看 [基于Wsl Ubuntu20.04上安装Kuberneters](/posts/micro/kind-k8s-ubuntu2004-wsl)
使用wget或直接浏览器下载istio安装文件 [istio-1.8.2](https://github.com/istio/istio/releases/tag/1.8.2)
```sh
$ curl -L https://istio.io/downloadIstio | sh -
```
or
```sh
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.8.2 TARGET_ARCH=x86_64 sh -
```
我自己安装的是1.8.2版本
我是把压缩包下载回来了,之后解压,并移动目录
### 安装目录包含如下内容：
install/kubernetes 目录下，有 Kubernetes 相关的 YAML 安装文件
samples/ 目录下，有示例应用程序
bin/ 目录下，包含 istioctl 的客户端文件。istioctl 工具用于手动注入 Envoy sidecar 代理。
```sh
$ tar zxvf istio-1.8.2-linux-amd64.tar.gz
$ mv istio-1.8.2 ~/.local/
$ mkdir ~/.bin
$ ln -s ~/.local/istio-1.8.2/bin/istioctl ~/.bin/istioctl # 软链接
$ echo export PATH=$PATH:$HOME/.bin >> ~/.zshrc
$ source ~/.zshrc
$ istioctl version
no running Istio pods in "istio-system"
1.8.2
```
这样,就把istioctl工具安装好了.
### 安装demo到k8s
![](/images/posts/k8s/istio-profile.webp)
```sh
$ istioctl install
```
or
```sh
$ istioctl install --set profile=demo -y
```
安装需要等一段时间，由于都是在国外网站下载，有可能会超时失败，如果超时失败多试几次，应该会成功。
```
✔ Istio core installed                                                      
✔ Istiod installed                                                          
✔ Egress gateways installed                                                 
✔ Ingress gateways installed                                                   
✔ Installation complete 

$ kubectl label namespace default istio-injection=enabled
namespace/default labeled
```
### 发布一个示例
```sh
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created

$ kubectl get services
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.96.233.137   <none>        9080/TCP   3m45s
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    24m
productpage   ClusterIP   10.96.234.131   <none>        9080/TCP   3m44s
ratings       ClusterIP   10.96.216.255   <none>        9080/TCP   3m45s
reviews       ClusterIP   10.96.166.58    <none>        9080/TCP   3m45s

# 根据网速情况不同，大约等5分钟左右，pods将拉取完镜像并启动起来
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79c697d759-gzlmp       2/2     Running   0          4m21s
productpage-v1-65576bb7bf-qhjqc   1/2     Running   0          4m19s
ratings-v1-7d99676f7f-7zb5d       2/2     Running   0          4m21s
reviews-v1-987d495c-lw5wn         2/2     Running   0          4m20s
reviews-v2-6c5bf657cf-5jknp       2/2     Running   0          4m21s
reviews-v3-5f7b9f4f77-625fh       2/2     Running   0          4m20s

$ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -s productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```
