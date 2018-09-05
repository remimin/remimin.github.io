## cgroup简介

cgroups（Control Groups）最初叫Process Container，由Google工程师（Paul Menage和Rohit Seth）于2006年提出，
后来因为Container有多重含义容易引起误解，就在2007年更名为Control Groups，并被整合进Linux内核。顾名思义就是把进程放到一个组里面统一加以控制。

cgroups 的全称是control groups，cgroups为每种可以控制的资源定义了一个子系统。典型的子系统介绍如下：
- cpu 子系统，主要限制进程的 cpu 使用率。
- cpuacct 子系统，可以统计 cgroups 中的进程的 cpu 使用报告。
- cpuset 子系统，可以为 cgroups 中的进程分配单独的 cpu 节点或者内存节点。
- memory 子系统，可以限制进程的 memory 使用量。
- blkio 子系统，可以限制进程的块设备io。
- devices 子系统，可以控制进程能够访问某些设备。
- net_cls 子系统，可以标记cgroups中进程的网络数据包，然后可以使用tc 模块（traffic control）对数据包进行控制。
- freezer 子系统，可以挂起或者恢复cgroups中的进程。
- ns 子系统，可以使不同 cgroups 下面的进程使用不同的 namespace。

docker中对Cgroup的使用

. mount -t tmpfs cgroups /sys/fs/cgroup.

## Linux 查看是否支持cgroup
查看/boot/config-* 文件中的CGROUP是否为y
```commandline
# grep CGROUP /boot/config-3.10.0-327.el7.x86_64
CONFIG_CGROUPS=y
# CONFIG_CGROUP_DEBUG is not set
CONFIG_CGROUP_FREEZER=y
CONFIG_CGROUP_DEVICE=y
CONFIG_CGROUP_CPUACCT=y
CONFIG_CGROUP_HUGETLB=y
CONFIG_CGROUP_PERF=y
CONFIG_CGROUP_SCHED=y
CONFIG_BLK_CGROUP=y
# CONFIG_DEBUG_BLK_CGROUP is not set
CONFIG_NETFILTER_XT_MATCH_CGROUP=m
CONFIG_NET_CLS_CGROUP=y
CONFIG_NETPRIO_CGROUP=m
```