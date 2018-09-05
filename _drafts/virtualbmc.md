--
layout:     post
title:      "Qemu调试虚拟机问题"
subtitle:   ""
date:       2018-05-10
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - impi, ironic, virtualization
---

# Virtualbmc安装
```commandline
pip install virtualbmc
```
为了防止python依赖包冲突问题可以使用virtualenv 
`mkvirtualenv impi`

# Virtualbmc的使用

## 创建虚拟机
可以使用virsh命令自己定义虚拟机，也可以使用nova来创建
```commandline
 nova boot --image bf508b95-27bd-4f73-90ed-cde00e2929e2 --flavor 7 --nic net-id=cc49211f-5c59-497f-a825-ec161fad0875 2C_vm
 +--------------------------------------+-----------------------------------------------+
| Property                             | Value                                         |
+--------------------------------------+-----------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                        |
| OS-EXT-AZ:availability_zone          |                                               |
| OS-EXT-SRV-ATTR:host                 | -                                             |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                             |
| OS-EXT-SRV-ATTR:instance_name        | instance-000000c2                             |
| OS-EXT-STS:power_state               | 0                                             |
| OS-EXT-STS:task_state                | scheduling                                    |
| OS-EXT-STS:vm_state                  | building                                      |
| OS-SRV-USG:launched_at               | -                                             |
| OS-SRV-USG:terminated_at             | -                                             |
| accessIPv4                           |                                               |
| accessIPv6                           |                                               |
| adminPass                            | UVdhdG6pGHLq                                  |
| config_drive                         |                                               |
| created                              | 2018-06-01T08:02:04Z                          |
| flavor                               | 2C2G (7)                                      |
| hostId                               |                                               |
| id                                   | 8ae39a67-d674-4117-90f1-919e37b7c012          |
| image                                | cirros (bf508b95-27bd-4f73-90ed-cde00e2929e2) |
| key_name                             | -                                             |
| metadata                             | {}                                            |
| name                                 | 2C_vm                                         |
| os-extended-volumes:volumes_attached | []                                            |
| progress                             | 0                                             |
| security_groups                      | default                                       |
| status                               | BUILD                                         |
| tenant_id                            | dbdd1e7b27884c39ae19194b587a83c5              |
| updated                              | 2018-06-01T08:02:04Z                          |
| user_id                              | e5121dc28c3e47daae754b8f0457c618              |
+--------------------------------------+-----------------------------------------------+
```

1 ) vbmc查看管理的虚拟机
```commandline
vbmc list
```
2 ) 添加虚拟机到vbmc管理，port可以自主选择

```commandline
vbmc add instance-000000c2  --port 623
```

3 ) 再次查看vbmc list
```commandline
 vbmc list
+-------------------+--------+---------+------+
|    Domain name    | Status | Address | Port |
+-------------------+--------+---------+------+
| instance-000000c2 |  down  |    ::   | 623  |
+-------------------+--------+---------+------+

```
4 ) 启动添加的虚拟机

```commandline
vbmc start instance-000000c2

vbmc list
+-------------------+---------+---------+------+
|    Domain name    |  Status | Address | Port |
+-------------------+---------+---------+------+
| instance-000000c2 | running |    ::   | 623  |
+-------------------+---------+---------+------+

```

5 ) 测试ipmi命令

```commandline
(impi) [root@server-31 ~]#  ipmitool -I lanplus -U root -P calvin -H 127.0.0.1 power status
Chassis Power is on
(impi) [root@server-31 ~]#  ipmitool -I lanplus -U root -P calvin -H 127.0.0.1 power status 623
Chassis Power is on
```

ironic create node
```commandline
[root@server-31 polex]# ironic node-create -d pxe_ipmitool_socat
+-------------------+--------------------------------------+
| Property          | Value                                |
+-------------------+--------------------------------------+
| chassis_uuid      |                                      |
| driver            | pxe_ipmitool_socat                   |
| driver_info       | {}                                   |
| extra             | {}                                   |
| name              | None                                 |
| network_interface |                                      |
| properties        | {}                                   |
| resource_class    |                                      |
| uuid              | 2fcf3708-8dc4-4373-9d0f-7cc435dd336f |
+-------------------+--------------------------------------+
[root@server-31 polex]# ironic node-list
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| 2fcf3708-8dc4-4373-9d0f-7cc435dd336f | None | None          | None        | available          | False       |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+

```

手动更新 ironic node的管理信息

```commandline
 nova boot --image a5ebf960-ee23-4748-98c8-656741e36cf8 --nic net-name=provision-net  --flavor 2C4G bm-001-vbmc
+--------------------------------------+----------------------------------------------------------+
| Property                             | Value                                                    |
+--------------------------------------+----------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                   |
| OS-EXT-AZ:availability_zone          |                                                          |
| OS-EXT-SRV-ATTR:host                 | -                                                        |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                                        |
| OS-EXT-SRV-ATTR:instance_name        | instance-000000c6                                        |
| OS-EXT-STS:power_state               | 0                                                        |
| OS-EXT-STS:task_state                | scheduling                                               |
| OS-EXT-STS:vm_state                  | building                                                 |
| OS-SRV-USG:launched_at               | -                                                        |
| OS-SRV-USG:terminated_at             | -                                                        |
| accessIPv4                           |                                                          |
| accessIPv6                           |                                                          |
| adminPass                            | ^Polex234                                                |
| config_drive                         |                                                          |
| created                              | 2018-06-04T09:17:26Z                                     |
| flavor                               | 2C4G (8)                                                 |
| hostId                               |                                                          |
| id                                   | f7b4d8c9-cce8-418c-8b52-8a6f124e8645                     |
| image                                | centos7.2_product (a5ebf960-ee23-4748-98c8-656741e36cf8) |
| key_name                             | -                                                        |
| metadata                             | {}                                                       |
| name                                 | bm-001-vbmc                                              |
| os-extended-volumes:volumes_attached | []                                                       |
| progress                             | 0                                                        |
| security_groups                      | default                                                  |
| status                               | BUILD                                                    |
| tenant_id                            | dbdd1e7b27884c39ae19194b587a83c5                         |
| updated                              | 2018-06-04T09:17:27Z                                     |
| user_id                              | e5121dc28c3e47daae754b8f0457c618                         |
+--------------------------------------+----------------------------------------------------------+

```

```commandline
[root@server-31 ~]# vbmc add instance-000000c6 --port 624 --username admin --password admin
[root@server-31 ~]# vbmc list
+-------------------+---------+---------+------+
|    Domain name    |  Status | Address | Port |
+-------------------+---------+---------+------+
| instance-000000c2 | running |    ::   | 623  |
| instance-000000c6 |   down  |    ::   | 624  |
+-------------------+---------+---------+------+

[root@server-31 ~]# vbmc start instance-000000c6
[root@server-31 ~]# 2018-06-04 17:21:24,863.863 193394 INFO VirtualBMC [-] Virtual BMC for domain instance-000000c6 started

[root@server-31 ~]# vbmc list
+-------------------+---------+---------+------+
|    Domain name    |  Status | Address | Port |
+-------------------+---------+---------+------+
| instance-000000c2 | running |    ::   | 623  |
| instance-000000c6 | running |    ::   | 624  |
+-------------------+---------+---------+------+

(impi) [root@server-31 ~]# ipmitool -I lanplus -U admin -P admin -H 127.0.0.1 -p 623 power status
Chassis Power is on

```

```commandline
ironic node-update 2fcf3708-8dc4-4373-9d0f-7cc435dd336f  add driver_info/ipmi_username=admin driver_info/ipmi_password=admin driver_info/ipmi_address="172.24.5.1" driver_info/impi_port=623
```

# 手动部署网络

ovs-vsctl add-br br-ironic

修改  /etc/neutron/plugins/ml2/openvswitch_agent.ini
bridge_mappings = physnet3:br-ex,ironicnet:br-ironic

重启
systemctl restart neutron-server
systemctl restart neutron-openvswitch-agent
systemctl restart neutron-dhcp-agent
systemctl restart neutron-l3-agent

创建flat provisioning network

```commandline
neutron net-create  --provider:physical_network  ironicnet --provider:network_type flat --port_security_enabled=false provision-net
neutron subnet-create 6540bdbb-a05b-4bf1-b993-8c1e6888d9ea 192.168.104.0/24 --disable-dhcp
```


问题记录：

1. provision network dhcp 应该是disabled
2. 在ironic 服务内部启动dnsmasq 用于ironic-inspector
3. 虚拟机内部要把ironic provioning network 的
4. deploy kernel & ramdisk 最小内存是4G
5. 当ironic node处于maintenance为true时， nova-compute resource tracker report 0
6. ironic.conf 下列配置很重要，必须配置为provision网络可达的
[conductor]
api_url=http://192.168.104.4:6385   # baremetal node MUST can access this address
force_power_state_during_sync = false
automated_clean = false
service_url=http://192.168.104.4:5050
7.  dd的问题，当时用vxlan的时候，dd经常卡住，应该vxlan mtu的问题，修改为flat问题就不存在了
8. port 映射的问题还是有，需要了解ironic-conductor的port映射机制
9. capabilites boot_option 原理 boot device 原理
10. dd 操作时占用cpu 和内存 会导致ironic服务卡
11. baremetal port 的问题

controller 节点
scheduler_use_baremetal_filters=true
scheduler_host_manager=nova.scheduler.host_manager.IronicHostManage

# 设置在完成第一次部署之后，重启采用本地启动
nova flavor-key FLAVOR_NAME set capabilities:boot_option="local"


## MTU 导致iscsi 挂载dd失败
原因分析：
物理机的网卡MTU为1500，而部署出来的ironic 服务的虚拟机的MTU为1450(vxlan overlay network)，导致dd命令通过
iscsi写入数据时发生了数据包的切割

## inspect 阶段拿不到ip的原因
1. dnsmasq 的eth0 必须要配置ip地址
2. neutron port-update --no-security-group 
neutron port-update  --port_security_enabled=false
或者neutron net-create --port_security_enabled=false

## ironic 中的hashring
ironic/common/hash_ring.py:HashRingManager

## Provision network

```commandline
neutron net-create provision-net --provider:network_type flat --provider:physical_network ironicnet
```

查看镜像

```commandline
virt-filesystems --long --parts --blkdevs -h -a centos-7.qcow2
Name       Type       MBR  Size  Parent
/dev/sda1  partition  83   8.0G  /dev/sda
/dev/sda   device     -    8.0G  -
```

guestmount 虚拟机镜像，使用yum命令安装rpm包
```commandline
 yum --disableexcludes=all \
 --setopt=cachedir=/tmp/image/var/cache/yum/x86_64/7/openstack-queens \
 --setopt=reposdir=/tmp/image/etc/yum.repos.d  \
 --installroot /tmp/image install ironic

```