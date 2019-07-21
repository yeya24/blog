---
title: "使用kubeadm安装k8s"
date: 2019-04-06T16:15:35+08:00
draft: false
description: ""
tags:
- "Kubernetes"
categories: 
- "Kubernetes"
---
至少使用2台节点，当然最好3台，每台资源达到4核8G，所使用的环境在Centos 7.5版本及以上

事先准备：给每台节点改主机名称，在k8s中需要每一台机器有一个独特的主机名，在centos里面使用hostnamectl这个工具来更改
``` bash
hostnamectl set-hostname xxx
reboot -n # 这一步是重启
```

安装docker，安装完成之后自行检查docker ps 或者docker run检查安装情况
``` bash
yum update -y
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum makecache fast
yum install -y docker-ce
systemctl enable docker
systemctl restart docker
```

在每台机器上运行下面的一个脚本，具体方法是创建一个setting.sh文件 chmod +x setting.sh  将脚本的内容复制进去，后运行脚本即可
```
#!/bin/sh
# 关闭防火墙
systemctl stop firewalld && systemctl disable firewalld

# 关闭selinux
setenforce 0
sed -i  's/SELINUX=enforcing/SELINUX=disabled/g'  /etc/selinux/config

# 关闭交换分区
swapoff -a && sysctl -w vm.swappiness=0
sed '/swap.img/d' -i /etc/fstab

# 添加阿里云的k8s源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 添加需要用到的内核网络参数
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf
```

每台机器执行完上面的脚本之后，再创建一个脚本 image.sh，将以下内容复制进去，之后 chmod +x image.sh这个脚本以后可以用来拉取镜像
``` bash
#!/bin/sh
curl -s https://zhangguanzhang.github.io/bash/pull.sh | bash -s -- $1
```

接下来如果是master节点，创建一个install-master.sh脚本并运行
``` bash
#!/bin/sh
version=1.13.2
yum install -y kubelet-$version
yum install -y kubectl-$version kubeadm-$version 
systemctl enable kubelet
systemctl restart kubelet

cat <<EOF > ~/kubeadm.conf
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
networking:
  podSubnet: "192.168.0.0/16"
kubernetesVersion: "v1.13.2"
EOF

~/image.sh k8s.gcr.io/kube-controller-manager:v$version
~/image.sh k8s.gcr.io/kube-proxy:v$version
~/image.sh k8s.gcr.io/kube-apiserver:v$version
~/image.sh k8s.gcr.io/kube-scheduler:v$version
~/image.sh k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
~/image.sh k8s.gcr.io/coredns:1.2.6
~/image.sh k8s.gcr.io/etcd:3.2.24
~/image.sh k8s.gcr.io/pause:3.1
docker pull calico/node:v3.6.0
docker pull calico/cni:v3.6.0
docker pull calico/kube-controllers:v3.6.0
```

如果是node，创建一个install-node.sh脚本并运行
``` bash
#!/bin/sh
version=1.13.2
yum install -y kubelet-$version
yum install -y kubeadm-$version 
systemctl enable kubelet
systemctl restart kubelet

~/image.sh k8s.gcr.io/kube-proxy:v$version
~/image.sh k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
~/image.sh k8s.gcr.io/coredns:1.2.6
~/image.sh k8s.gcr.io/pause:3.1
docker pull calico/node:v3.6.0
docker pull calico/cni:v3.6.0
docker pull calico/kube-controllers:v3.6.0
```

等待每个节点上面的脚本都执行完成之后，在master节点上运行下面的命令正式开始安装
``` bash
kubeadm init --config kubeadm.conf
```

当出现运行成功的字样之后，执行命令
``` bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

然后在node节点上面执行安装完成后跳出来的命令
以下面的这行命令为例，在master节点上安装完成之后下面会显示出类似的这么一条命令，你只要复制下来在node节点上运行一下即可（下面的只是个示例）
``` bash
kubeadm join 10.14.10.139:6443 --token zxqopw.kkoe6hduf6113pmn --discovery-token-ca-cert-hash sha256:42de13e61ba1d1647b4f9b21fcf05964b826f84df0db41f87c0b66fb48bb2d32
```

最后的步骤是安装calico，calico是我这边选择的网络组件，下面的命令都在master上面执行
``` bash
kubectl taint node k8s-m1 node-role.kubernetes.io/master- # 这一步是去除master上面的污点，这个地方要注意，将我前面的 k8s-m1 换成你那边对应的master节点的名称，实际情况下这步可以不用执行，非必须。

wget https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
kubectl apply -f calico.yaml
```

可以通过kubectl get nodes 查看状态，一般来说出现下面的状态就是都ok了，如果不ok，一般来说可能是网络插件还没启动或者运行出错。
``` bash
NAME     STATUS   ROLES    AGE     VERSION
k8s-m1   Ready    master   4d23h   v1.13.2
k8s-m2   Ready    <none>   4d23h   v1.13.2
k8s-m3   Ready    <none>   4d23h   v1.13.2
```
