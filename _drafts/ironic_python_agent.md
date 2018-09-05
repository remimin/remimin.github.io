

# PXE inspection
PXE需要网卡支持, PXE 客户机(client)这个术语是指机器在PXE启动过程中的角色。一个PXE 客户机(client)可以是一台服务器、桌面级计算机、笔记本电脑或者其他装有PXE启动代码的机器。`
# DHCP 阶段
DHCP是基于udp协议，source port 为68， server port为67，baremetal node启动时，PXE client发送DHCP Discover 广播，通过tcpdump 抓包如下:
```commandline
04:08:22.570308 IP (tos 0x0, ttl 64, id 341, offset 0, flags [none], proto UDP (17), length 424)
    0.0.0.0.bootpc > 255.255.255.255.bootps: [udp sum ok] BOOTP/DHCP, Request from fa:16:5e:ec:78:e6, length 396, xid 0x96cf6856, secs 4, Flags [none] (0x0000)
          Client-Ethernet-Address fa:16:5e:ec:78:e6
          Vendor-rfc1048 Extensions
            Magic Cookie 0x63825363
            DHCP-Message Option 53, length 1: Discover
            MSZ Option 57, length 2: 1472
            ARCH Option 93, length 2: 0
            NDI Option 94, length 3: 1.2.1
            Vendor-Class Option 60, length 32: "PXEClient:Arch:00000:UNDI:002001"
            CLASS Option 77, length 4: "iPXE"
            Parameter-Request Option 55, length 22:
              Subnet-Mask, Default-Gateway, Domain-Name-Server, LOG
              Hostname, Domain-Name, RP, Vendor-Option
              Vendor-Class, TFTP, BF, Option 119
              Option 128, Option 129, Option 130, Option 131
              Option 132, Option 133, Option 134, Option 135
              Option 175, Option 203
            T175 Option 175, length 45: 177.5.1.26.244.16.0.235.3.1.0.0.23.1.1.34.1.1.19.1.1.17.1.1.39.1.1.25.1.1.16.1.2.33.1.1.21.1.1.24.1.1.18.1.1
            Client-ID Option 61, length 7: ether fa:16:5e:ec:78:e6
            GUID Option 97, length 17: 0.201.216.180.247.232.204.140.65.139.82.138.111.18.78.134.85
            END Option 255, length 0
```
Option 53: 表示dhcp request 类型
Option 60: VCI, 供应商Class识别（VCI Vendor Class Identifier）client使用这个option提供硬件信息
Option 55: dhcp client请求的信息



ironic node状态
- enroll

录入状态，即通过ironic api，创建了node关联了具体的物理机器，单并未管理

- managable 

将node标记为"managable"，表明ironic已经进行处理的管理接口检测通过。例如node是ipmi_pxe driver的，那么
表明ipmi信息录入正确

- verifying


- inspecting

Hardware 信息收集中

- available

已经可以进行调度分配了

- active

Node已经成功部署

- wait call-back

在provision阶段的中间状态，等待ipa启动进行回调

- deploying

在provision阶段的中间状态，ipa回调成功后，node会进入deploying阶段，进行具体的部署操作，如磁盘分区，操作系统image拷贝等

- deploy failed

部署失败

- deploy complete

部署操作完成

- cleaning





```commandline
# file bm-deploy.ramdisk
bm-deploy.ramdisk: gzip compressed data, from Unix, last modified: Wed Mar  8 07:27:17 2017
```
可见ramdisk是gzip压缩文件，可以使用ungzip进行ramdisk 解压
```commandline
mkdir unpack
cd unpack
gzip -dc /path/to/the/ramdisk | cpio -id
```

```commandline
find . | cpio -H newc -o > /path/to/the/new/ramdisk
```

参数传递

/proc/cmdline

# TFTP server 搭建

1. 创建boot loader 目录
```commandline
# Bios 
mkdir -p /var/lib/tftpboot/pxelinux/pxelinux.cfg
# Uefi
mkdir -p /var/lib/tftpboot/efi
```

2. 拷贝boot loader 文件
下载grub2和shim安装文件，并解压
```commandline
rpm2cpio grub2-efi-version.rpm | cpio -idmv 
rpm2cpio shim-version.rpm | cpio -idmv
cp ./boot/efi/EFI/centos/grubx64.efi /var/lib/tftpboot/efi
cp ./boot/efi/EFI/centos/shim.efi /var/lib/tftpboot/efi
```
