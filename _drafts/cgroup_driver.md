## systemd vs. 

### 概念
- service: 通过systemd管理配置文件定义的一组进程

- scope: 被fork出来的子进程构成，同一组子进程会被放到同一个scope下，scope是systemd进程扫描系统进程得到的

- slice: 不包含进程，但会组件一个层级。scope和service都会属于某个slice层级下。系统默认有四种slice
    - _.slice: 根slice
    - system.slice: 所有service的slice
    - user.slice： 用户会话slice
    - machine.slice: 所有虚拟机和LXC slice

cgroup vs slice



## kubelet cgroup drivers

{"kubepods", "burstable", "pod1234-abcd-5678-efgh"}

/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod1234_abcd_5678_efgh.slice

## cgroupfs 

"/" + path.Join(cgroupName...)