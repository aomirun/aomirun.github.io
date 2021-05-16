---
title: "基于Wsl Ubuntu20.04上安装Kuberneters"
date: 2021-02-04T13:50:02+08:00
draft: false
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
---

## 前言
win10可谓windows系统对开发人员最友好的版本了,linux命令行加win的桌面系统,强强联合,正因为这个Linux子系统,让本地测试环境搭建与开发起飞.
下面就针对在wsl2下面的ubuntu20.04上面安装k8s开发环境做点说明.
创建本地k8s开发环境,主要是通过kind来实现

## kind 安装
kind（Kubernetes IN Docker） 是一个基于 docker 构建 Kubernetes 集群的工具，非常适合用来在本地搭建基于 Kubernetes 的开发/测试环境。所以在安装kind前,最好把wsl,docker,国内镜像这些前置工作准备好.
通过 https://github.com/kubernetes-sigs/kind/releases/latest 获取最新的 release，根据自己的系统类型选择相应的系统 release，下载 release 之后重命名为 kind（Windows 系统 release 重命名为 kind.exe)，然后将其放在某一个目录下，并要确保这个目录在系统 PATH 中以方便的使用，可以放在 usr/bin 目录下（默认已经在系统 PATH 中），linux 系统中可能需要配置文件权限
```sh
$ sudo chmod +x kind
```
or 
需要本地有golang环境,并且GOPATH/bin在PATH中,如果不在,需要 `export PATH="$(go env GOPATH)/bin:${PATH}"`
```sh
$ GO111MODULE=on go get sigs.k8s.io/kind@v0.10.0
```
## 拉取镜像  kindest/node 
最新的 v0.20.2/v0.19.7 我测试时有bug,后来测试到v.19.1是正常的,所以这里我就采用了v1.19.1
```sh
$ docker pull kindest/node:v1.19.1
```
## 操作 Kuberneters 集群
使用 kind 创建 Kubernetes 集群非常的方便，只需要一行命令即可
```sh
$ kind create cluster #这里默认是使用kind的默认最新版本,我测试新版本有问题,故我测试的时候指定了下面的版本
```
or 特定版本
```sh
$ kind create cluster --image kindest/node:v1.19.1
```
### 成功的返回
```sh
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.19.1) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```
删除集群
```sh
$ kind delete cluster
```
默认集群名称是 "kind"，如果要创建多个或者指定集群名称，可以指定 name 参数：
```sh
$ kind create cluster --name=k8s-cluster1
```
删除集群
```sh
kind delete cluster --name=k8s-cluster1
```

## kubectl 安装
```sh
$ sudo apt update 
$ sudo apt upgrade
$ sudo apt install -y apt-transport-https
$ curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
$ echo "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
$ sudo apt update
$ sudo apt install -y kubelet kubeadm kubectl
```
创建集群成功之后，就可以使用 kubectl 来操作 k8s 集群了
## 查看集群信息
```sh
$ kubectl get all --all-namespaces
NAMESPACE            NAME                                             READY   STATUS    RESTARTS   AGE
kube-system          pod/coredns-f9fd979d6-sg289                      1/1     Running   0          6m43s
kube-system          pod/coredns-f9fd979d6-wjksf                      1/1     Running   0          6m43s
kube-system          pod/etcd-kind-control-plane                      1/1     Running   0          6m50s
kube-system          pod/kindnet-9jv9s                                1/1     Running   0          6m44s
kube-system          pod/kube-apiserver-kind-control-plane            1/1     Running   0          6m50s
kube-system          pod/kube-controller-manager-kind-control-plane   1/1     Running   0          6m50s
kube-system          pod/kube-proxy-5fmtx                             1/1     Running   0          6m44s
kube-system          pod/kube-scheduler-kind-control-plane            1/1     Running   0          6m50s
local-path-storage   pod/local-path-provisioner-78776bfc44-fqm22      1/1     Running   0          6m43s

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  7m3s
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   7m2s

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kindnet      1         1         1       1            1           <none>                   7m1s
kube-system   daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   7m2s

NAMESPACE            NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
kube-system          deployment.apps/coredns                  2/2     2            2           7m2s
local-path-storage   deployment.apps/local-path-provisioner   1/1     1            1           7m1s

NAMESPACE            NAME                                                DESIRED   CURRENT   READY   AGE
kube-system          replicaset.apps/coredns-f9fd979d6                   2         2         2       6m44s
local-path-storage   replicaset.apps/local-path-provisioner-78776bfc44   1         1         1       6m44s
```
## 查看容器内部容器列表
获取这个容器内部的运行容器列表，这个容器内部通过 crictl 来操作容器，可以参考 https://github.com/kubernetes-sigs/cri-tools
```sh
$ docker exec kind-control-plane crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID
03496f54d192d       bfe3a36ebd252       8 minutes ago       Running             coredns                   0                   e170783fe655f
f2874e6f6eba7       bfe3a36ebd252       8 minutes ago       Running             coredns                   0                   f2d4a12df3a93
178d427a80a68       e422121c9c5f9       8 minutes ago       Running             local-path-provisioner    0                   1600c76f4bc03
d019fbe2d252a       b77790820d015       8 minutes ago       Running             kindnet-cni               0                   f94e9d590d2ea
d8bc5b5bf64ea       47e289e332426       8 minutes ago       Running             kube-proxy                0                   84ff49f7a0532
a2e91623770af       0369cf4303ffd       8 minutes ago       Running             etcd                      0                   d680864effc4a
753ad7868585a       7dafbafe72c90       8 minutes ago       Running             kube-controller-manager   0                   c0325d328da3d
a246720f380f2       4d648fc900179       8 minutes ago       Running             kube-scheduler            0                   af718164f1b76
7a499ab5ae3f2       8cba89a89aaa8       8 minutes ago       Running             kube-apiserver            0                   735ec1dcd8891
```
kind 创建集群成功之后，就可以向 kubernetes 集群部署资源了，开始你的 Kubernetes 之旅吧~
## Kind：创建仪表板
```sh
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```
执行上面命令如果碰到了如下情况,就到[github](https://github.com/kubernetes/dashboard/blob/master/aio/deploy/recommended.yaml)上面看源代码,自己本地建一个来使用.
```
The connection to the server raw.githubusercontent.com was refused - did you specify the right host or port?
```
新建一个用户kind放脚本的目录
```sh
$ mkdir ~/kind && cd ~/kind && touch recommended.yaml
# 把内容复制到recommended.yaml中,然后执行下面的命令

$ kubectl apply -f recommended.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```
### 创建dashboard-adminuser.yaml
```sh
$ cat > dashboard-adminuser.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard  
EOF
```
### 创建登录用户
```sh
$ kubectl apply -f dashboard-adminuser.yaml
```
说明：上面创建了一个叫admin-user的服务账号，并放在kubernetes-dashboard 命名空间下，并将cluster-admin角色绑定到admin-user账户，这样admin-user账户就有了管理员的权限。默认情况下，kubeadm创建集群时已经创建了cluster-admin角色，我们直接绑定即可。
### 查看admin-user账户的token
```sh
$ kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```
### 创建本地代理
```sh
$ kubectl proxy
```
现在即可使用浏览器打开
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
输入刚刚获取到的token,点登录,即可进入.
![](/images/posts/k8s/k8s-dashboard.png)
## 安装helm
```sh
$ curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -   
$ sudo apt install apt-transport-https --yes
$ echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
$ sudo apt update
$ sudo apt install helm
$ helm version
version.BuildInfo{Version:"v3.5.2", GitCommit:"167aac70832d3a384f65f9745335e9fb40169dc2", GitTreeState:"dirty", GoVersion:"go1.15.7"}
```
## 安装ingress-nginx
这里会用到k8s.gcr.io的docker镜像，请用中科大镜像加速
```sh
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm repo update

$ helm install ingress-nginx ingress-nginx/ingress-nginx
```
## 结语
至此，k8s的本地开发环境就创建好了，后续就是往里加东西了。
