---
layout:     post
title:      "kolla部署单节点ceph"
subtitle:   ""
date:       2018-01-04
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - kolla
    - ceph
---
# kolla部署单节点ceph

> 按照ceph推荐部署最少是需要3个节点，但作为开发测试环境资源有限只能部署单个节点，之前参照过ceph官方使用ceph-depoly部署过单节点ceph，这次尝试使用kolla部署一下，顺便趟一下kolla的坑

---

## 准备阶段
1. 安装ansible
```yum install ansible```
2. 安装docker, 参照官网

3. 创建systemctl配置文件
```
# 可以配置systemctl 配置文件管理docker服务
mkdir -p /etc/systemd/system/docker.service.d
tee /etc/systemd/system/docker.service.d/kolla.conf <<-'EOF'
[Service]
MountFlags=shared
EOF
```
4. 重启动docker服务
```
systemctl daemon-reload
systemctl restart docker
```
5. 准备物理磁盘先进行分区
```
parted /dev/sde -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP 1 -1
```
6. 获取kolla-ansible最新代码
```
git clone https://github.com/openstack/kolla-ansible.git
```
7. 安装kolla-ansible
```
cd kolla-ansible
pip install -rrequirements.txt
pip install -e ./ #development install
mkdir -p /usr/share/kolla-ansible
cp -r ansible/ /usr/share/kolla-ansible
```

## Kolla部署ceph all-in-one
#### 1. ceph配置
```
mkdir -p /etc/kolla/config
cp -r kolla-ansible/etc/kolla /etc/kolla/
tee  /etc/kolla/config/ceph.conf <<-"EOF"
[global]
osd pool default size = 1
osd pool default min size = 1
EOF
```
#### 2. 全局配置
增加如下配置项，ceph rgw 可以根据自己的需求决定是否配置
```
/etc/kolla/globals.yml 
openstack_release: master
enable_ceph: "yes"
# enable_ceph_rgw: "no"
```
#### 3. 生成password
/etc/kolla/password.yml
如果不需要部署openstack的话,其实这步不需要执行`kolla-genpwd`，仅仅添加
docker_registry_password: null 即可
#### 4. 运行部署
```
cd kolla-ansible/ansible/inventory
kolla-ansible deploy -i ./all-in-one
```

#### 5. 部署完成，确认ceph状态
```
docker exec -it ceph_mon ceph -s
  cluster:
    id:     756d71a5-70eb-4586-bc2f-6aed24dea531
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum 127.0.0.1
    mgr: localhost(active)
    osd: 3 osds: 3 up, 3 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 bytes
    usage:   321 MB used, 1101 GB / 1102 GB avail
    pgs:
```

### 查看ceph containers
![](/img/kolla-ceph-contains.png)
- ceph_mon: ceph monitor 服务
- ceph_mgr: ceph manager 服务
- ceph_osd_*: ceph osd 服务

### ceph常用命令
```
docker exec -it ceph_mon ceph -s
docker exec -it ceph_mon ceph osd pool ls
docker exec -it ceph_mon ceph osd pool set vms size 1
docker exec -it ceph_mon ceph osd pool set images size 1
docker exec -it ceph_mon ceph osd pool set volumes size 1
docker exec -it ceph_mon ceph osd pool set backups size 1
docker exec -it ceph_mon ceph osd pool set rbd size 1
docker exec -it ceph_mon ceph osd pool set rbd min_size 1
docker exec -it ceph_mon ceph osd pool set backups min_size 1
docker exec -it ceph_mon ceph osd pool set volumes min_size 1
docker exec -it ceph_mon ceph osd pool set images min_size 1
docker exec -it ceph_mon ceph osd pool set vms min_size 1
```


## 如何在container外部使用ceph

- 安装ceph的cli
- 查看ceph-mon的ceph配置文件路径

```
# docker volume ls
DRIVER              VOLUME NAME
local               ceph_mon
local               ceph_mon_config
local               kolla_logs

#  docker volume inspect ceph_mon_config
[
    {
        "CreatedAt": "2018-01-08T11:39:16+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/ceph_mon_config/_data",
        "Name": "ceph_mon_config",
        "Options": {},
        "Scope": "local"
    }
]

```
- 拷贝ceph的配置文件到本地
```
cp -r /var/lib/docker/volumes/ceph_mon_config/_data /etc/ceph
```

## 遇到的问题
#### docker升级docker-ce
我的操作系统是centos 7.2， 原始安装的docker版本是1.10.*，但是在下载docker image时总是碰到unkown EOF的问题，即使是使用了代理。
查看了官方文档，于是docker进行升级于是碰到了docker-runc无法启动的问题，具体的描述参考[docker-runc bug](https://github.com/moby/moby/issues/35906)，还好有人提供了workaround 方法`yum install http://mirror.centos.org/centos/7/os/x86_64/Packages/libseccomp-2.3.1-3.el7.x86_64.rpm` 重启docker即可，根据讨论内容，docker-ce官方其实是支持centos7.4的版本。

#### koll-ansible默认设置
kolla-ansible 默认是开启了很多openstack的服务，如果不需要，得在`/etc/kolla/global.yml`中覆盖掉默认的变量值
默认开始的变量定义在`kolla-ansible/ansible/group_vars/all.yml`中
其中enable_开头的变量即为是否部署对应服务
所有kolla支持的部署服务见[官网](https://docs.openstack.org/kolla-ansible/latest/reference/index.html)

#### ceph数据清除
如果仅部署了ceph， 可以统统`kolla-ansible destroy` 删除
想要重新部署ceph，需要首先删除container和所有相关的volume,并对磁盘重新进行格式化, 并删除/etc/fstabs里面的container, 删除/etc/kolla下ceph的相关配置
```
systemctl daemon-reload
```

## 总结
遇到的问题都是docker相关的以及对配置文件不熟悉，配置文件改正确之后，非常顺利的部署了allinone的环境，用作开发测试还是很值得推荐的，不需要对ceph做很多的配置工作。

### 参考
- [https://docs.openstack.org/kolla-ansible/latest/reference/ceph-guide.html](https://docs.openstack.org/kolla-ansible/latest/reference/ceph-guide.html)
- [https://docs.docker.com/engine/reference/commandline/cli/](https://docs.docker.com/engine/reference/commandline/cli/)
- [Docker CE installation in CentOS](https://docs.docker.com/engine/installation/linux/docker-ce/centos/)