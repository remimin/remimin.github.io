---
layout:     post
title:      "k8s device plugin"
subtitle:   ""
date:       2018-07-06
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - k8s
    - openshift
---

## kubernetes

## etcd

### Raft协议

### watch 机制

### api server 缓存机制


### scheduler
master/slave 
只连当前节点的api server

选主算法：`kubectl get ep -n kube-system -o yaml` 


pod & node state

minion list：
- 调用pod list 生成
- informor factory ?
predicates: 预选

priorities: 优选


QA: 如何保证并发资源计算一致性？

### controller manager

状态机
deployment state
会根据定义去检测pod状态及策略

### kubelet

pod来源：
- server：master
- file：配置文件 

工作流：
kubelet-> CRI -> docker-shim -> docker -<grpc>-> containerd -<rrpc>-> docker-shim -> runc ->sys
kubelet -> CRI -> runtime
 
 问题：
 - cgroupfs
 - systemd
 
 
 
 ### kube-proxy
 

 dc: deploymentconfig
 rc: replication controller
 
 
# Openshift

harbor: image
nfs


## 亲和和反亲和
效率还比较低


# CNI

