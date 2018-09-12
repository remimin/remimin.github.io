---
layout:     post
title:      "以容器方式运行虚拟机"
subtitle:   ""
date:       2018-09-10
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - k8s
    - container
    - kubevirt
---

# 以容器方式运行虚拟机

## kubevirt安装

virt-launcher调用libvirt创建虚拟机，即每个虚拟机会对应对立的libvirtd

虚拟机的root disk 生成方式两种：
1. registryDisk 有virt-handler将下载的image转为raw格式，然后启动virt-launcher将image作为root disk
2. 使用pvc(persistentvolumeclaim)，要求存在`/disk.img`文件

代码分析可见`virt-launcher/virtwrap/converter.go` volume支持的disk type均为file类型


```commandline
# kubectl get vmis
NAME               AGE
vmi-flavor-small   15h
vmi-windows        14h
```

#### kubevirt 虚拟机网络bridge


veth536682 和 container 内部的eth0是veth peer

```
# host bridge
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024215192e92       no              veth53dd982

# virt-launcher container linux bridge
bridge name     bridge id               STP enabled     interfaces
br1             8000.02420a4cebaa       no              eth0
                                                        vnet0
# vnet0虚拟机网卡
    <interface type='bridge'>
      <mac address='02:42:0a:0a:0b:06'/>
      <source bridge='br1'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </interface>
```

## 问题记录

- privileged forbidden
```text
The DaemonSet "virt-handler" is invalid: spec.template.spec.containers[0].securityContext.privileged: Forbidden: disallowed by cluster policy
```

解决方法`exprot ALLOW_PRIVILEGED=true` 重启`hack/local-up-cluster.sh`

```
# cluster/kubectl.sh get pods -n kube-system
NAME                              READY     STATUS    RESTARTS   AGE
virt-api-747ff7bdc6-fbll8         1/1       Running   0          5m
virt-api-747ff7bdc6-ss86g         1/1       Running   1          5m
virt-controller-767587575-m6224   0/1       Running   4          5m
virt-controller-767587575-r5nr6   0/1       Running   3          5m
virt-handler-wq7lr                1/1       Running   0          5m
```

virt-launcher pod

```commandline
Command:
      /usr/share/kubevirt/virt-launcher/entrypoint.sh
      --qemu-timeout
      5m
      --name
      vmi-flavor-small
      --namespace
      default
      --kubevirt-share-dir
      /var/run/kubevirt
      --readiness-file
      /tmp/healthy
      --grace-period-seconds
      15
      --hook-sidecars
      0
```



- virt-api x509认证失败

原因 KUBE_ENABLE_CLUSTER_DNS=false

- allinone 模式 kubernetes_provider=local virt-controller failed

坑的原因，都是因为代理惹得祸，因为设置了代理给kube cluster导致的 

livenessprobe实际上是kubelet直接检测pod的状态，因为加了代理，导致检测失败


问题记录： k8s namespace ???


evel=error timestamp=2018-07-30T03:52:14.951323Z pos=server.go:68 component=virt-launcher namespace=default name=vmi-flavor-small-pvc kind= uid=ba6ae222-93ab-11e8-b1d2-246e96275bc0 reason="virError(Code=38, Domain=18, Message='Cannot access storage file '/var/run/kubevirt-private/vmi-disks/rbdvolume/disk.img': No such file or directory')" msg="Failed to sync vmi"



## kubernetes rbd volume

workflow： rbd map 将image映射到device，然后mount到指定的mount point，attach 给pod



## virtlet 
### installation
```commandline
go get -u -d github.com/Mirantis/criproxy github.com/Mirantis/virtlet

bash -x build/cmd.sh build

kubectl label node kubdev --overwrite extraRuntime=virtlet
kubectl create configmap -n kube-system virtlet-image-translations --from-file  `pwd`/deploy/images.yaml


docker exec virtlet-build "${remote_project_dir}/_output/virtletctl" gen "--dev" >virtlet.yaml

```