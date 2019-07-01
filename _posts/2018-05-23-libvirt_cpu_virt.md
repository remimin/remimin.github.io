---
layout:     post
title:      "KVM虚拟化之host capability"
subtitle:   ""
date:       2018-05-23
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - kvm, virtualization
---
# KVM虚拟化之host capability

KVM是Linux内核的一个模块，基于硬件虚拟化技术实现VMM的功能。KVM主要是通过操作与处理器共享的数据结构来实现指令集以及MMU的虚拟化，
捕捉Guest的IO指令(包括Port IO和mmap IO)以及实现中断虚拟化。至于IO设备的软件模拟，是通过用户程序QEMU来实现的。
QEMU负责解释IO指令流，并将其请求换成系统调用或者库函数传给Host操作系统，让Host上的驱动去完成真正的IO操作。
使用KVM技术创建虚拟机，首先需要了解宿主机(host)的虚拟化能力，此文章就是介绍宿主机的虚拟化能力各项指标。

## host capabilities

可以使用`virsh`或者其他编程接口命令获取宿主机的Capabilites。Capabilities内容包含两部分host和guest。

host部分包含的:
- uuid #SMBIOS UUID， 也可以通过libvirtd.conf host_uuid覆盖这个值
- cpu
    - arch  #主机架构
    - model # cpu model，不同的cpu model有不同更多的指令集和features
    - vendor # cpu 厂商
    - microcode # 
    - topology # cpu 拓扑
        - sockets
        - cores
        - threads
    - features # cpu Features，没有cpu model所覆盖到的features
    - pages
- power_management #是否支持memory suspend, disk hibernation or hybrid suspend
- migration_features 
- topology #CPU物理拓扑
    - cell #NUMA Node
    - memory # NUMA Node Memory
    - pages
    - distances
    - cpus   #processor
- secmodel   #安全设置

### CPU Topology

对于cpu topology中sockets是主板上CPU的槽数也常被称为“路”，core即经常说的核数, threads就是每个core上可以并发的线程数目，
即超线程。通过`lscpu`命令或者`cat /proc/cpuinfo`可以看到更多cpu细节参数。

当CPU的sockets是2，cores数为12，threads为2，那么processor（通常说的逻辑cpu）的数量为2*12*2即48，也有人称为48核CPU。
在虚拟化中，48即为物理宿主机的vcpu总数目，即在未超额配置情况下最大可分配的vcpu数量。

1) 查看物理机的sockets数目 
    ```commandline
     grep 'physical id' /proc/cpuinfo | awk -F: '{print $2 | "sort -un"}'
    ```
    
2) 查看每个Socket有几个Processor
    ```commandline
    grep 'physical id' /proc/cpuinfo | awk -F: '{print $2}' | sort | uniq -c
    ```
    

## Guest Capability

- os_type  #支持使用的hypervisor 类型
- arch  # Guest Arch
- features #
    - pae  #32位guest使用的PAE内存地址管理
    - nopae #32位guest不使用pae地址管理
    - ia64_be  #IA64 guest 大端寻址模式
    - acpi  #电源管理模块，默认是启用的，toggle属性用于定义是否允许guest覆盖
    - apic #是否支持中断控制器虚拟化
    - cpuselection # 支持定义cpu分配
    - deviceboot #支持定义boot device
    - disksnapshot #支持外部定义的snapshot 

### Guest os_type

capabilites中<guest>字段中提供的是libvirt可以用来启动guest vm的driver类型。Libvirt中定义Guest虚拟化类型的xml字段就是os_type

 

| Driver | Guest Type |
| :--------| :----------|
| qemu | Always "hvm" |
| xen | Either "xen" for a paravirtualized guest or "hvm" for a fully virtualized guest |
| uml | Always "uml" |
| lxc | Always "exe" |
| vbox | Always "hvm" |
| openvz | Always "exe" |
| one | Always "hvm" |
| ex | Not supported at this time |

更多描述详见：[capability information](https://libvirt.org/guide/html/Application_Development_Guide-Connections-Capability_Info.html)


## Guest CPU mode
通过libvirt定义guest时，需要指定cpu mode, cpu mode有以下几种:
- "host-model"，默认是这种模式。
    在这种mode下，libvirt 根据当前宿主机 CPU 指令集从配置文件
    `/usr/share/libvirt/cpu_map.xml`**自动选择**一种最相配的CPU型号。虚拟机的指令集往往比宿主机少，
    性能相对host-passthrough要差，好处是**在迁移时，它允许目的宿主机CPU和原宿主机的存在一定的差异**。

- "host-passthrough"
    这种模式完全将宿主得cpu mode透传给guest vm，这种模式下vm就具有了宿主机cpu的能力，这种模式下，
    虚拟机迁移只能在相同类型cpu的宿主机之间进行迁移。host-passthrough 模式需要hypervisor的支持
    在kvm中必须开启虚拟化嵌套
   
- "custom"
    当定义为custom模式时，必须要定义cpu model，可选增加cpu Features，宿主机支持的cpu model可以查询
    `/usr/share/libvirt/cpu_map.xml`

要获取CPU全部支持的features，可以通过`virsh cpu-baseline`获取，或者是`cat /proc/cpuinfo |grep flags`
Features是用于判断是否兼容的关键，当虚拟机想要迁移到其他宿主机时，可以通过`virsh cpu-compare`命令检测是否兼容。
    

### 开启嵌套虚拟化

1. 打开kvm内核模块nested特性
`modprobe kvm-intel nested=1`
或者修改modprobe.d 编辑`/etc/modprobe.d/kvm_mod.conf`，添加以下内容
```
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1
```

```commandline
modprobe -r kvm-intel
modprobe -a kvm-intel
```

检查是否打开nested功能
```
$ cat /sys/module/kvm_intel/parameters/nested 
Y
```



虚拟机xml配置cpu mode为host-passthrough，即要将物理机CPU特性全部传给虚拟机
```xml
&lt cpu mode='host-passthrough'\&gt
```

### APIC介绍
Intel处理器提供高级可编程中断控制器的硬件虚拟化（APICv，Advanced Programmable
Interrupt Controller）。APIC虚拟化将通过允许guest vm直接访问APIC改善虚拟化x86_64 guest vm性能，
大幅减少中断等待时间和高级可编程中断控制器造成的虚拟机退出数量。更新的Intel处理器中默认使用此功能，并可改善I/O性能



