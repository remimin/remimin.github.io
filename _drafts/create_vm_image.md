---
layout:     post
title:      "修改虚拟机镜像"
subtitle:   ""
date:       2018-05-10
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - image, kvm, virtualization
---
有时你拿到一个虚拟机镜像，但你在将它上传到 OpenStack服务之前，想要修改镜像里的内容。这里介绍几个可以修改镜像的工具。

## guestfish

guestfish并不直接mount镜像文件到本地文件系统，而是提供一个shell接口，你可以通过这个shell接口对镜像内文件做查看，编辑，删除操作，诸如 touch, chmod, 和 rm的 guestfish 命令，就像普通bash命令一样。
 
假设你有一个文件名为centos63_desktop.img的 CentOS qcow2 格式的虚拟机镜像。用root用户挂载这个镜像为可读可写模式，如下：

```commandline
# guestfish --rw -a centos63_desktop.img

Welcome to guestfish, the libguestfs filesystem interactive shell for
editing virtual machine filesystems.

Type: 'help' for help on commands
      'man' to read the manual
      'quit' to quit the shell

><fs>

```
在做任何操作之前，必须先在 guestfish提示符运行run命令。它会启动一个虚拟机，用于完成后续操作。
```commandline

><fs> run
 100% ⟦▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒⟧ 00:00
><fs>

```

使用virsh命令可以查看创建的虚拟机guestfs-* 

```commandline
[root@server-31 ~]# virsh list
 Id    Name                           State
----------------------------------------------------
 66    guestfs-wazfbvp9ihrz7hl4       running
```

通过list-filesystems命令，我们可查看镜像内的文件系统列表：
```commandline
><fs> list-filesystems
/dev/sda1: xfs


```

挂载包含根分区的那个逻辑卷，然后执行操作
```commandline
><fs> mount /dev/sda1 /
><fs>ls /usr/lib/python2.7/site-packages/ |grep ironic
```




