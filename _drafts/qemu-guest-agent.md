---
layout:     post
title:      "vhost"
subtitle:   ""
date:       2018-05-11
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - kvm, virtualization, qga
---

# Linux vhost介绍

virtio是一种IO半虚拟化方案，是guest和host所约定的一种协议用来减少guest IO时的VM-Exit（guest和host的上下文切换）
并且使guest和host能并行处理IO来提高throughput和减少latency。通常来说，virtio的数据都会在guest和hypervisor间转发，
这就导致了在数据交换的时候有多次的上文切换。例如guest在发包到外部网络的情况下，首先guest需要切换到host kernel，
然后host kernel会切换到hyperisor来处理guest的请求，hypervisor通过系统调用将数据包发送到外部网络后切换回host kernel，然后再切换回guest。
这样漫长的路径无疑会带来性能上的损失。vhost就在这样的背景下产生了。它是位于host kernel的一个模块，用于和guest直接通信，
Linux kernel中的vhost driver提供了KVM/Qemu在kernel环境中的virtio设备的模拟，通过设备模拟代码可以直接进入host kernel子系统，
从而不需要从用户空间通过系统调用陷入内核，减少了由于模拟IO导致的性能下降。 

Linux kernel vhost的代码都在drivers/vhost下。
vhost由多种driver:
- vhost-net (linux/drivers/net.c) virtio-net
- vhost-scsi (linux/drivers/scsi.c) virtio-scsi
- vhost-sock (linux/drivers/vsock.c) virtio-serial
- vhost-vring (linux/drivers/vringh.c)

## vhost-net 驱动模型
vhost-net driver创建了一个字符设备 /dev/vhost-net，这个设备可以被用户空间打开，并可以被ioctl命令操作。当给一个Qemu进程传递了参数
`-netdev tap,vhost=on`的时候，QEMU会通过调用几个ioctl命令对这个文件描述符进行一些初始化的工作，然后进行特性的协商，
从而宿主机跟客户机的vhost-net driver建立关系。

在vhost driver初始化阶段创建了一个kernel thread，命名规则是vhost-$pid， pid是qemu进程的pid，这个kernel thread的主要任务就是处理I/O
events和执行设备模拟。

vhost没有p

QEMU代码调用如下：

`net_init_tap ->net_init_tap_one ->vhost_net_init-> -> vhost_dev_init -> vhost_net_ack_features`

在`vhost_net_init`中调用了`vhost_dev_init`，打开`/dev/vhost-net`这个设备，然后返回一个文件描述符作为`vhost-net`的后端，
`vhost_dev_init`调用的ioctl命令有
`r = ioctl(hdev->control, VHOST_SET_OWNER, NULL);`
Kernel 中的定义为：
```
1.  /* Set current process as the (exclusive) owner of this file descriptor.  This  
2.   * must be called before any other vhost command.  Further calls to  
3.   * VHOST_OWNER_SET fail until VHOST_OWNER_RESET is called. */  
4.  #define VHOST_SET_OWNER _IO(VHOST_VIRTIO, 0x01)
```

然后获取VHOST支持的特性
```
r = ioctl(hdev->control, VHOST_GET_FEATURES, &features);
同样，kernel中的定义为：

1.  /* Features bitmask for forward compatibility.  Transport bits are used for  
2.   * vhost specific features. */  
3.  #define VHOST_GET_FEATURES      _IOR(VHOST_VIRTIO, 0x00, __u64)  
```

QEMU中用vhost_net 这个数据结构代表打开的vhost_net 实例：
```
1.  struct vhost_net {  
2.      struct vhost_dev dev;  
3.      struct vhost_virtqueue vqs[2];  
4.      int backend;  
5.      NetClientState *nc;  
6.  };  
```
```
struct vhost_dev {
	struct mm_struct *mm;
	struct mutex mutex;
	struct vhost_virtqueue **vqs;
	int nvqs;
	struct eventfd_ctx *log_ctx;
	struct llist_head work_list;
	struct task_struct *worker;
	struct vhost_umem *umem;
	struct vhost_umem *iotlb;
	spinlock_t iotlb_lock;
	struct list_head read_list;
	struct list_head pending_list;
	wait_queue_head_t wait;
};
```

使用ioctl设置完后，QEMU注册memory_listener 回调函数:
```
1.  hdev->memory_listener = (MemoryListener) {  
2.      .begin = vhost_begin,  
3.      .commit = vhost_commit,  
4.      .region_add = vhost_region_add,  
5.      .region_del = vhost_region_del,  
6.      .region_nop = vhost_region_nop,  
7.      .log_start = vhost_log_start,  
8.      .log_stop = vhost_log_stop,  
9.      .log_sync = vhost_log_sync,  
10.     .log_global_start = vhost_log_global_start,  
11.     .log_global_stop = vhost_log_global_stop,  
12.     .eventfd_add = vhost_eventfd_add,  
13.     .eventfd_del = vhost_eventfd_del,  
14.     .priority = 10  
15. };  
```
## Kernel 中的virtio 模拟
vhost并没有完全模拟一个pci设备，相反，它只把自己限制在对virtqueue的操作上。worker thread 一直等待virtqueue的数据，
对于vhost-net来说，当virtqueue的tx队列中有数据了，它会把数据传送到与其相关联的tap设备文件描述符上。相反，worker thread 
也要进行tap文件描述符的轮询，对于vhost-net，当tap文件描述符有数据到来时候，worker thread会被唤醒，然后将数据传送到rx队列中。

## vhost的在用户空间的接口
数据已经准备好了，如何通知客户机呢？
从vhost模块的依赖性可以得知，vhost这个模块并没有依赖于kvm模块，也就是说，理论上其他应用只要实现了和vhost接口，也可以调用vhost来进行数据传输的加速。
但也正是因为vhost跟kvm模块没什么关系，当QEMU（KVM）guest把数据放到tx队列（virtqueue）上之后，它是没有办法直接通知vhost数据准备了的。

不过，vhost设置了一个eventfd文件描述符，这个文件描述符被前面我们提到的worker thread监控，所以QEMU可以通过向eventfd发送消息告诉vhost数据准备好了。
QEMU的做法是这样的，在QEMU中注册一个ioeventfd，当guest发生I/O退出了，会被KVM捕捉到，KVM向vhost发送eventfd从而告知
vhost KVM guest已经准备好数据了。由于 worker thread监控这个eventfd，在收到消息后，知道guest已经把数据放到了tx队列，可以进行对戏

vhost通过发出一个guest中断，通过KVM提供的irqevent，告诉guest需要传送的buffer已经放到了rx virtqueue了，QEMU（KVM）注册这个irq PCI事件，
得知内核空间的数据准备好了，调用guest驱动进行数据的读取。

所以，总的来说 vhost实例需要知道的有三样

- guest的内存映射，也就是virtqueue，用于数据的传输
- qemu kick eventfd，vhost接收guest发送的消息，该消息被worker thread捕获
- call event（irqfd）用于通知guest
以上三点在QEMU初始化的时候准备好，数据的传输只在内核空间就完成了，不需要QEMU进行干预，所以这也是为什么使用vhost进行传输数据高效的原因。

### Guest vhost 数据访问路径
Put the datapath into kernel side to reduce context switch
Sequences

– a) VHOST_SET_OWNER: set the owner of vhost fd.
– b) VHOST_SET_MEM_TABLE: tell the guest memory info to vhost.
– c) VHOST_SET_VRING_NUM: set the vqueue size
– d) VHOST_SET_VRING_BASE: the base position of avail queue.
– e) VHOST_SET_VRING_ADDR: the address of descriptor, avail queue,
used queue.
– f) VHOST_SET_VRING_CALL: set the eventfd to inject interrupt to guest
– g) VHOST_NET_SET_BACKEND: set backed socket or tap fd.
– h) VHOST_SET_VRING_KICK: set the evntfd to monitor vqueue notify
signal. 

# vhost-user 介绍
在 vhost 的方案中，由于 vhost 实现在内核中，guest 与 vhost 的通信，相较于原生的 virtio 方式性能上有了一定程度的提升，从 guest 到 kvm.ko
的交互只有一次用户态的切换以及数据拷贝。这个方案对于不同 host 之间的通信，或者 guest 到 host nic 之间的通信是比较好的，但是对于某些用户态进程间的通信，
比如数据面的通信方案，openvswitch 和与之类似的 SDN 的解决方案，guest 需要和 host 用户态的 vswitch 进行数据交换，如果采用 vhost 的方案，
guest 和 host 之间又存在多次的上下文切换和数据拷贝，为了避免这种情况，业界就想出将 vhost 从内核态移到用户态。这就是 vhost-user 的实现。
