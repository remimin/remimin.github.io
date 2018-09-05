---
layout:     post
title:      "minikube安装记录"
subtitle:   ""
date:       2018-07-06
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - k8s
---
# minikube安装记录

> 本文是Linux Ubuntu虚拟机内部使用none driver安装minikube

## 1. 安装docker-ce并配置docker代理
- [ubuntu install docker-ce](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
- [docker proxy configure](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)


## 配置HTTP/HTTPS代理

感谢万恶的墙，只有再祖国才有机会体会。注意一定要配置no_proxy，否则minikube安装后期访问kubeapiservers就会出错了，
除非你的代理是反向代理到你的虚拟机

```commandline
set_proxy(){
  export HTTP_PROXY=http://10.71.84.78:1080
  export HTTPS_PROXY=http://10.71.84.78:1080
  export NO_PROXY="localhost"
}
```

## minikube 安装


```commandline
minikube start --vm-driver none  --alsologtostderr --v 7
```
使用none driver安装，加上日志输入，能清楚看到minikube安装的步骤。

通过`jourctl -ef -u kubelet`查看日志输入，能看到具体错误出现在了哪个过程

通过命令行输出，可以看到minikube主要是调用`kubeadm init`命令去创建一个极简k8s环境的。
```commandline
I0706 00:25:41.627148   43345 exec_runner.go:50] Run with output: sudo /usr/bin/kubeadm init --config /var/lib/kubeadm.yaml --ignore-preflight-errors=DirAvailable--etc-kubernetes-manifests --ignore-preflight-errors=DirAvailable--data-minikube --ignore-preflight-errors=Port-10250 --ignore-preflight-errors=FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml --ignore-preflight-errors=FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml --ignore-preflight-errors=FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml --ignore-preflight-errors=FileAvailable--etc-kubernetes-manifests-etcd.yaml --ignore-preflight-errors=Swap --ignore-preflight-errors=CRI
```

## etcd bind出错

使用docker logs $id 查看etcd出错的原因：

```commandline

2018-07-06 06:15:38.384384 I | etcdmain: etcd Version: 3.1.12
2018-07-06 06:15:38.384435 I | etcdmain: Git SHA: 918698add
2018-07-06 06:15:38.384439 I | etcdmain: Go Version: go1.8.7
2018-07-06 06:15:38.384442 I | etcdmain: Go OS/Arch: linux/amd64
2018-07-06 06:15:38.384448 I | etcdmain: setting maximum number of CPUs to 1, total number of available CPUs is 1
2018-07-06 06:15:38.384496 I | embed: peerTLS: cert = /var/lib/localkube/certs/etcd/peer.crt, key = /var/lib/localkube/certs/etcd/peer.key, ca = , trusted-ca = /var/lib/localkube/certs/etcd/ca.crt, client-cert-auth = true
2018-07-06 06:15:38.384502 W | embed: The scheme of peer url http://localhost:2380 is HTTP while peer key/cert files are presented. Ignored peer key/cert files.
2018-07-06 06:15:38.384506 W | embed: The scheme of peer url http://localhost:2380 is HTTP while client cert auth (--peer-client-cert-auth) is enabled. Ignored client cert auth for this url.
2018-07-06 06:15:38.445310 C | etcdmain: listen tcp 10.68.146.116:2380: bind: cannot assign requested address
```
发现etcd listen的时候总是到一个未知的地址
[k8s issue 57709](https://github.com/kubernetes/kubernetes/issues/57709)中讨论到
ubuntu 16.04的dns解析是跳过了本地，虽然我虚拟机上直接用`ping localhost`是正常的，但是使用
`nslookup localhost`则是解析到了另外一个地址。
一个简单的做法就是把/etc/resolve.conf里的search注释掉。

```commandline
# kubectl get namespaces
NAME          STATUS    AGE
default       Active    2h
kube-public   Active    2h
kube-system   Active    2h
```

```commandline
# kubectl --namespace kube-system get pods
NAME                                    READY     STATUS    RESTARTS   AGE
etcd-minikube                           1/1       Running   0          2h
kube-addon-manager-minikube             1/1       Running   0          2h
kube-apiserver-minikube                 1/1       Running   0          2h
kube-controller-manager-minikube        1/1       Running   0          2h
kube-dns-86f4d74b45-dd7q5               3/3       Running   1          2h
kube-proxy-jgqtf                        1/1       Running   0          2h
kube-scheduler-minikube                 1/1       Running   0          2h
kubernetes-dashboard-5498ccf677-47mtd   1/1       Running   0          2h
storage-provisioner                     1/1       Running   0          2h
```

```commandline
# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /root/.minikube/ca.crt
    server: https://10.71.84.87:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /root/.minikube/client.crt
    client-key: /root/.minikube/client.key
```
