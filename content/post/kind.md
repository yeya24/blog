---
title: "使用kind来快速部署k8s环境"
date: 2019-04-05T10:23:15+08:00
draft: false
description: "kind"
tags:
- "Kubernetes"
categories: 
- "Kubernetes"
---
## 啥是kind
[kind](https://github.com/kubernetes-sigs/kind) 即 Kubernetes In Docker，顾名思义，就是将 k8s 所需要的所有组件，全部部署在一个docker容器中，是一套开箱即用的 k8s 环境搭建方案。使用 kind 搭建的集群无法在生产中使用，但是如果你只是想在本地简单的玩玩 k8s，不想占用太多的资源，那么使用 kind 是你不错的选择。同样，kind 还可以很方便的帮你本地的 k8s 源代码打成对应的镜像，方便测试。
## 使用kind
在学校的一台 centos 上简单尝试一下 kind，前提是必须要安装好 docker 和 kubectl。

``` 
wget https://github.com/kubernetes-sigs/kind/releases/download/0.2.1/kind-linux-amd64
mv kind-linux-amd64 kind
chmod +x kind
mv kind /usr/local/bin
```

安装完成之后，我们来看看 kind 有哪些命令。

build 用来从 k8s source 构建一个镜像。

create、delete 创建、删除集群。

export 命令目前只有一个 logs 选项，作用是将内部所有容器的日志拷贝到宿主机的某个目录下。

get 查看当前有哪些集群，哪些节点，以及 kubectl 配置文件的地址

load 可以从宿主机向 k8s 容器内导入镜像。

```
[root@node-2 ~]# kind
kind creates and manages local Kubernetes clusters using Docker container 'nodes'

Usage:
  kind [command]

Available Commands:
  build       Build one of [base-image, node-image]
  create      Creates one of [cluster]
  delete      Deletes one of [cluster]
  export      exports one of [logs]
  get         Gets one of [clusters, nodes, kubeconfig-path]
  help        Help about any command
  load        Loads images into nodes
  version     prints the kind CLI version

Flags:
  -h, --help              help for kind
      --loglevel string   logrus log level [panic, fatal, error, warning, info, debug] (default "warning")
      --version           version for kind

Use "kind [command] --help" for more information about a command.
```

下面来以最简单的方式安装一个集群

```
[root@node-2 ~]# kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.13.4) 🖼
 ✓ Preparing nodes 📦 
 ✓ Creating kubeadm config 📜 
 ✓ Starting control-plane 🕹️ 
Cluster creation complete. You can now use the cluster with:

export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
kubectl cluster-info

[root@node-2 ~]# export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
[root@node-2 ~]# kubectl cluster-info
Kubernetes master is running at https://localhost:39284
KubeDNS is running at https://localhost:39284/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[root@node-2 ~]# kubectl get node
NAME                 STATUS   ROLES    AGE   VERSION
kind-control-plane   Ready    master   99s   v1.13.4
```

使用 **kind create cluster** 安装，是没有指定任何配置文件的安装方式。从安装打印出的输出来看，分为4步：

1. 查看本地上是否存在一个基础的安装镜像，默认是 kindest/node:v1.13.4，这个镜像里面包含了需要安装的所有东西，包括了 kubectl、kubeadm、kubelet 二进制文件，以及安装对应版本 k8s 所需要的镜像，都以 tar 压缩包的形式放在镜像内的一个路径下
2. 准备你的 node，这里就是做一些启动容器、解压镜像之类的工作
3. 生成对应的 kubeadm 的配置，之后通过 kubeadm 安装，安装之后还会做另外的一些操作，比如像我刚才仅安装单节点的集群，会帮你删掉 master 节点上的污点，否则对于没有容忍的 pod 无法部署。
4. 启动完毕

查看当前集群的运行情况

```
[root@node-2 ~]# kubectl get po -n kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-86c58d9df4-6g66f                     1/1     Running   0          21m
coredns-86c58d9df4-pqcc4                     1/1     Running   0          21m
etcd-kind-control-plane                      1/1     Running   0          20m
kube-apiserver-kind-control-plane            1/1     Running   0          20m
kube-controller-manager-kind-control-plane   1/1     Running   0          20m
kube-proxy-cjgnt                             1/1     Running   0          21m
kube-scheduler-kind-control-plane            1/1     Running   0          21m
weave-net-ls2v8                              2/2     Running   1          21m
```

默认方式启动的节点类型是 control-plane 类型，包含了所有的组件。包括2 * coredns、etcd、api-server、controller-manager、kube-proxy、sheduler，网络插件方面默认使用的是 weave，且目前只支持 weave，不支持其他配置，如果需要可以修改 kind 代码进行定制。

基本上，kind 的所有秘密都在那个基础镜像中。下面是基础容器内部的 /kind 目录，在 bin 目录下安装了 kubelet、kubeadm、kubectl 这些二进制文件，images 下面是镜像的 tar 包，kind 在启动基础镜像后会执行一遍 docker load 操作将这些 tar 包导入。manifests 下面是 weave 的 cni。

```
root@kind-control-plane:/kind# ls
bin  images  kubeadm.conf  manifests  systemd  version

root@kind-control-plane:/kind# ls bin/
kubeadm  kubectl  kubelet

root@kind-control-plane:/kind# ls images/
4.tar  6.tar  8.tar               kube-controller-manager.tar  kube-scheduler.tar
5.tar  7.tar  kube-apiserver.tar  kube-proxy.tar

root@kind-control-plane:/kind# ls manifests/
default-cni.yaml

root@kind-control-plane:/kind# ls systemd/
10-kubeadm.conf  kubelet.service
```

## 创建多节点的集群
默认安装的集群只带上了一个控制节点，下面重新创建一个两节点的集群。配置文件如下，在 node 中可以配置的不是很多，除了 role 另外的可以更改 node 使用的镜像，不过我这边还是使用默认的镜像。

```
apiVersion: kind.sigs.k8s.io/v1alpha3
kind: Cluster
nodes:
  - role: control-plane
  - role: worker
```

还是使用命令安装

```
[root@node-2 ~]# kind create cluster --config=kind-config.yaml 
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.13.4) 🖼
 ✓ Preparing nodes 📦📦 
 ✓ Creating kubeadm config 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Joining worker nodes 🚜 
Cluster creation complete. You can now use the cluster with:

export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
kubectl cluster-info

[root@node-2 ~]# kubectl get nodes
NAME                 STATUS   ROLES    AGE     VERSION
kind-control-plane   Ready    master   3m20s   v1.13.4
kind-worker          Ready    <none>   3m8s    v1.13.4

[root@node-2 ~]# kubectl get po -n kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-86c58d9df4-cnqhc                     1/1     Running   0          5m29s
coredns-86c58d9df4-hn9mv                     1/1     Running   0          5m29s
etcd-kind-control-plane                      1/1     Running   0          4m24s
kube-apiserver-kind-control-plane            1/1     Running   0          4m17s
kube-controller-manager-kind-control-plane   1/1     Running   0          4m21s
kube-proxy-8t4xt                             1/1     Running   0          5m27s
kube-proxy-skd5v                             1/1     Running   0          5m29s
kube-scheduler-kind-control-plane            1/1     Running   0          4m18s
weave-net-nmfq2                              2/2     Running   1          5m27s
weave-net-srdfw                              2/2     Running   0          5m29s
```

大功告成，一套测试集群很轻松就搭建完成啦。
