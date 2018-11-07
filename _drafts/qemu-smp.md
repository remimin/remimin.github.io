# qemu之CPU
# SMP
使用qemu创建虚拟机，当cpu类型选用SMP时，通常需要指定参数`-smp 2,sockets=2,cores=1,threads=1`
表示申请虚拟机的vcpu数量为2。其中:
sockets: 表示虚拟机的vcpu硬件数量，对应物理其实是cpu芯片数量， 例子中为2
cores: 表示每个cpu上的cores即硬件线程数，例子中为1
threads: 表示超线程数，例子中为1

# NUMA
