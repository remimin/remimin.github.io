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

# OCI

## 微服务
- 服务发现
- 配置中心
- 服务安全
- API网关
- 负载均衡
- 服务自制容错
- 扩容缩容
- 日志中心
- 监控中心
- 服务链路跟踪
- 自动化部署
- 服务调动
- 资源管理
- 进程隔离

＝＝　微服务设计　＝＝

系统设计　－> 搭建技术架构 -> 编写业务代码 -> 

## istio

流量管理：细粒度的流量管理，提供断路器，超时，重试
安全：通信加密，黑白名单
可观察性：监控/日志/服务追踪
集成和定制：策略执行组件可以发展和定制，可与现有的ACL，日志，监控，配额，审计等方案集成
多平台支持：k8s, consul, 虚拟机

### 架构


Controll pannel

Pilot：Platform adapter, abstract model, envoy api
Mixer：权限管理和
Citadel: 

service proxy: envoy 
check and report: 
sidecar 形式注入微服务实例
拦截所有进出流量
依据策略进行管理

## oci