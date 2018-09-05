* [kata container](#kata-container)
    * [kata container 安装](#安装kata-container-runtime)
    * [kata container images](#准备kata-container-image)
* [k8s本地安装](#k8s本地安装)
* [CRI-O安装](#CRI-O安装)
* [kublet & kata-containers](#kubelet-与kata-container)


## kata container

kata containers是由OpenStack基金会管理，但独立于OpenStack项目之外的容器项目。
它是一个可以使用容器镜像以超轻量级虚机的形式创建容器的运行时工具。 kata containers整合了Intel的 Clear Containers 和 Hyper.sh 的 runV，
能够支持不同平台的硬件 （x86-64，arm等），并符合OCI(Open Container Initiative)规范。
目前项目包含几个配套组件，即Runtime，Agent， Proxy，Shim，Kernel等。目前Kata Containers的运行时还没有整合，
即Clear containers 和 runV还是独立的。

![k8s-kc](https://katacontainers.io/media/uploads/katacontainers/images/posts/kubernetesandkatacontainers.jpg)

> 环境信息
>
> os: CentOS Linux release 7.4.1708 (Core)
>
> docker 1.12.6
>
> etcd: 3.2.11
>
> go: 1.9.4
>
> kubenetes: 1.10.5


### 安装kata container runtime

获取源码并执行编译安装`kata-runtime kata-proxy kata-shim`
```commandline
go get -d -u github.com/kata-containers/runtime github.com/kata-containers/proxy github.com/kata-containers/shim
cd $GOPATH/src/github.com/kata-containers/runtime
make && make install
cd ${GOPATH}/src/github.com/kata-containers/proxy
make && make install 
cd ${GOPATH}/src/github.com/kata-containers/shim
make && make install 
```

运行`kata-check`检查环境是否满足kata container的要求，kata container要求宿主机具有硬件虚拟化的能力

```commandline
# kata-runtime kata-check
INFO[0000] CPU property found                            description="Intel Architecture CPU" name=GenuineIntel pid=156730 source=runtime type=attribute
INFO[0000] CPU property found                            description="Virtualization support" name=vmx pid=156730 source=runtime type=flag
INFO[0000] CPU property found                            description="64Bit CPU" name=lm pid=156730 source=runtime type=flag
INFO[0000] CPU property found                            description=SSE4.1 name=sse4_1 pid=156730 source=runtime type=flag
INFO[0000] kernel property found                         description="Host kernel accelerator for virtio network" name=vhost_net pid=156730 source=runtime type=module
INFO[0000] kernel property found                         description="Kernel-based Virtual Machine" name=kvm pid=156730 source=runtime type=module
INFO[0000] kernel property found                         description="Intel KVM" name=kvm_intel pid=156730 source=runtime type=module
WARN[0000] kernel module parameter has unexpected value  description="Intel KVM" expected=Y name=kvm_intel parameter=nested pid=156730 source=runtime type=module value=N
INFO[0000] Kernel property value correct                 description="Intel KVM" expected=Y name=kvm_intel parameter=unrestricted_guest pid=156730 source=runtime type=module value=Y
INFO[0000] kernel property found                         description="Host kernel accelerator for virtio" name=vhost pid=156730 source=runtime type=module
INFO[0000] System is capable of running Kata Containers  name=kata-runtime pid=156730 source=runtime
INFO[0000] device available                              check-type=full device=/dev/kvm name=kata-runtime pid=156730 source=runtime
INFO[0000] feature available                             check-type=full feature=create-vm name=kata-runtime pid=156730 source=runtime
INFO[0000] System can currently create Kata Containers   name=kata-runtime pid=156730 source=runtime

```

#### qemu-lite 安装

```commandline
$ source /etc/os-release
$ sudo yum -y install yum-utils
$ sudo -E VERSION_ID=$VERSION_ID yum-config-manager --add-repo "http://download.opensuse.org/repositories/home:/katacontainers:/release/CentOS_${VERSION_ID}/home:katacontainers:release.repo"
yum -y install qemu-lite
```

### 准备kata container image 

kata container image准备工作

#### initrd image


initrd(boot loader initialized RAM disk)就是由boot loader初始化时加载的ram disk。initrd是一个被压缩过的小型根目录，
这个目录中包含了启动阶段中必须的驱动模块，可执行文件和启动脚本。当系统启动的时候，booloader会把initrd文件读到内存中，
然后把initrd的起始地址告诉内核。内核在运行过程中会解压initrd，然后把 initrd挂载为根目录，然后执行根目录中的/initrc脚本，
您可以在这个脚本中运行initrd中的udevd，让它来自动加载设备驱动程序以及 在/dev目录下建立必要的设备节点。在udevd自动加载磁盘驱动程序之后，
就可以mount真正的根目录，并切换到这个根目录中。

#### 步骤

```commandline
 github.com/kata-containers/agent github.com/kata-containers/osbuilder

```

```commandline
# docker images
REPOSITORY                                    TAG                 IMAGE ID            CREATED             SIZE
image-builder-osbuilder                       latest              092d50027bf2        40 minutes ago      456.3 MB
centos-rootfs-osbuilder                       latest              27375c3d3491        About an hour ago   798.9 MB
```

#### rootfs image

1. 执行rootfs生成脚本，脚本执行完成后，能看到rootfs的文件夹 rootfs_Centos
```commandline
cd /root/.golang/src/github.com/kata-containers/osbuilder/rootfs-builder
export USE_DOCKER=true
./rootfs.sh centos
```
2. 执行rootfs image build 脚本

```commandline
cd /root/.golang/src/github.com/kata-containers/osbuilder/image-builder
image_builder.sh /root/.golang/src/github.com/kata-containers/osbuilder/rootfs-builder/rootfs-Centos
```

3. 执行initrd image build脚本

```commandline

```

#### image build细节

1. rootfs 生成

可以通过设置环境变量`exprot DEBUG=true`执行脚本，能看到更多的细节。rootfs.sh脚本目的就是生成distributor的根文件系统
在使用`USER_DOCKER=true`时，实际上是build一个centos-rootfs-osbuilder的docker image，然后从docker image创建一个
container，并在container内部执行rootfs.sh的脚本，把根文件系统导出来。

生成centos-root-osbuilder image如下：
```Dockerfile
From centos:7

RUN yum -y update && yum install -y git make gcc coreutils

# This will install the proper golang to build Kata components

RUN cd /tmp ; curl -OL https://storage.googleapis.com/golang/go1.9.2.linux-amd64.tar.gz
RUN tar -C /usr/ -xzf /tmp/go1.9.2.linux-amd64.tar.gz
ENV GOROOT=/usr/go
ENV PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```
```commandline
 docker run --rm --runtime runc --env https_proxy= --env http_proxy= 
 --env AGENT_VERSION=master --env ROOTFS_DIR=/rootfs --env GO_AGENT_PKG=github.com/kata-containers/agent 
 --env AGENT_BIN=kata-agent --env AGENT_INIT=no --env GOPATH=/root/.golang --env KERNEL_MODULES_DIR= 
 --env EXTRA_PKGS= --env OSBUILDER_VERSION=unknown 
 -v /root/.golang/src/github.com/kata-containers/osbuilder/rootfs-builder:/osbuilder 
 -v /root/.golang/src/github.com/kata-containers/osbuilder/rootfs-builder/rootfs-Centos:/rootfs 
 -v /root/.golang/src/github.com/kata-containers/osbuilder/rootfs-builder/../scripts:/scripts 
 -v /root/.golang/src/github.com/kata-containers/osbuilder/rootfs-builder/rootfs-Centos:/root/.golang/src/github.com/kata-containers/osbuilder/rootfs-builder/rootfs-Centos
 -v /root/.golang:/root/.golang centos-rootfs-osbuilder bash /osbuilder/rootfs.sh centos
```

2. rootfs image build
创建rootfs image的过程，简单来讲就是创建了一个raw格式的image，分区，拷贝rootfs的目录到分区内，脚本里root分区的文件系统是ext4

```commandline
qemu-img create -q -f raw "${IMAGE}" "${IMG_SIZE}M"
parted "${IMAGE}" --script "mklabel gpt" \
    "mkpart ${FS_TYPE} 1M -1M"
### ......    
cp -a "${ROOTFS}"/* ${MOUNT_DIR}
```
````commandline
docker run --rm --runtime runc --privileged --env IMG_SIZE= --env AGENT_INIT=no 
-v /dev:/dev -v /root/.golang/src/github.com/kata-containers/osbuilder/image-builder:/osbuilder 
-v /root/.golang/src/github.com/kata-containers/osbuilder/image-builder/../scripts:/scripts 
-v /root/.golang/src/github.com/kata-containers/osbuilder/rootfs-builder/rootfs-Centos:/rootfs 
-v /root/.golang/src/github.com/kata-containers/osbuilder/image-builder:/image image-builder-osbuilder 
bash /osbuilder/image_builder.sh -o /image/kata-containers.img /rootfs
````

3.  生成initrd image
生成initrd image的过程比较简单，主要是如下命令，其实就是把rootfs打包成initrd.img
````commandline
( cd "${ROOTFS}" && find . | cpio -H newc -o | gzip -9 ) > "${IMAGE_DIR}"/"${IMAGE_NAME}"
````



#### Guest Kernel image

## k8s 与kata container

cri-o是

kata container是hypverisor container阵营的container runtime项目，支持OCI标准。k8s想要创建kata container类型
pods需要的是cri shim即能够提供CRI的服务。k8s孵化项目CRI-O就是可以提供CRI并能够与满足OCI container runtime通讯的项目
k8s与kata container的work flow 如下

```text                                                                     +-----------+
                                +---------------+                      +--->|container  |
+---------------+               |  cri-o        |                      |    +-----------+
|  kubelet      |               |               |     +-------------+  |
| +-------------+ cri protobuf  +-------------+ |<--->|  container  +<-+    +-----------+
| | grpc client |<------------->| grpc server | |     |  runtime    +<----->|container  |
| +-------------|               +-------------+ |     +-------------+       +-----------+
+---------------+               |               |
                                +---------------+
```

- k8s调用kubelet在node上启动一个pod，kubelet通过gRPC调用cri-o启动pod。
- cri-o 使用`containers/image`从image registry获取image
- 调用`containers/stroage`将image解压成root filesystems
- cri-o根据kubelet api请求，创建OCI runtime spec文件
- cri-o调用container runtime(runc/kata container)创建container
- 每一个container都有一个`conmon`进程监控，用于处理container logs和exits code
- Pod网络CNI是直接调用了`CNI plugin`

cri-o 架构图

![](http://cri-o.io/assets/images/architecture.png)


查看conmon进程

conmon是cri-o启动的进程，看下crio的日志，可以看到当crio接收到容器创建请求时，会启动运行conmon命令


```commandline
running conmon: /usr/local/libexec/crio/conmon  args=[-c 783731ce2309dbfcb435a2ff47abf768d11916e886dc7a0c9b1f2f9d9fbeea9f -u
783731ce2309dbfcb435a2ff47abf768d11916e886dc7a0c9b1f2f9d9fbeea9f -r /usr/local/bin/kata-runtime -b /var/run/containers/storage/overlay-containers/783731ce2309dbfcb435a
2ff47abf768d11916e886dc7a0c9b1f2f9d9fbeea9f/userdata -p /var/run/containers/storage/overlay-containers/783731ce2309dbfcb435a2ff47abf768d11916e886dc7a0c9b1f2f9d9fbeea9f
/userdata/pidfile -l /var/log/pods/8b67a313-8a39-11e8-909c-246e96275bc0/783731ce2309dbfcb435a2ff47abf768d11916e886dc7a0c9b1f2f9d9fbeea9f.log --exit-dir /var/run/crio/e
xits --socket-dir-path /var/run/crio]

running conmon: /usr/local/libexec/crio/conmon  args=[-c 56d5fdeac760e903757db90da588cae7bb7a764baf4c2b2d49110114ba6d2baa -u
56d5fdeac760e903757db90da588cae7bb7a764baf4c2b2d49110114ba6d2baa -r /usr/local/bin/kata-runtime -b /var/run/containers/storage/overlay-containers/56d5fdeac760e903757db90da588cae7bb7a764baf4c2b2d49110114ba6d2baa/userdata -p /var/run/containers/storage/overlay-containers/56d5fdeac760e903757db90da588cae7bb7a764baf4c2b2d49110114ba6d2baa
/userdata/pidfile -l /var/log/pods/8b67a313-8a39-11e8-909c-246e96275bc0/nginx/0.log --exit-dir /var/run/crio/exits --socket-dir-path /var/run/crio]
```
  
```commandline
conmon -c 2ba098d0682c9f1623f52f18ea5320087ab9b252ee22c83b3fdf9ea45d789322 -u 2ba098d0682c9f1623f52f18ea5320087ab9b252ee22c83b3fdf9ea45d789322
  |-kata-proxy -listen-socket unix:///run/vc/sbs/2ba098d0682c9f1623f52f18ea5320087ab9b252ee22c83b3fdf9ea45d789322/proxy.sock -mux-socket/run
  |   `-7*[{kata-proxy}]
  |-kata-shim -agent unix:///run/vc/sbs/2ba098d0682c9f1623f52f18ea5320087ab9b252ee22c83b3fdf9ea45d789322/proxy.sock -container2ba098d0682c9f1
  |   `-8*[{kata-shim}]
  |-qemu-lite-syste -name sandbox-2ba098d0682c9f1623f52f18ea5320087ab9b252ee22c83b3fdf9ea45d789322 -uuid 83236965-4bac-4f32-a385-12c3c90c3d11 -machinepc,
  |   `-2*[{qemu-lite-syste}]
  `-{conmon}

conmon -c 45c1ee0637fdc33324edeb63f3b8eeaffed1e683cc1cfe9d32a45d178fbb658e -u 45c1ee0637fdc33324edeb63f3b8eeaffed1e683cc1cfe9d32a45d178fbb658e
  |-kata-shim -agent unix:///run/vc/sbs/2ba098d0682c9f1623f52f18ea5320087ab9b252ee22c83b3fdf9ea45d789322/proxy.sock -container45c1ee0637fdc33
  |   `-9*[{kata-shim}]
  `-{conmon}

```
  
  
问题记录：
1. iptables invalid mask 64, 重新build cni plugins 修复
2. group_manager = cgroupfs ,修改k8s cgroup_driver为cgroupfs

## k8s本地安装

```commandline
go get -d -u github.com/kubernetes/kubernetes
```

```commandline
cluster/kubectl.sh get pods
cluster/kubectl.sh get services

cluster/kubectl.sh run my-nginx --image=nginx --replicas=1 --port=80

# 查看kubernetes相关信息
cluster/kubectl.sh get pods
cluster/kubectl.sh get services
```


#### 


| | Alpine | CentOS | ClearLinux | EulerOS | Fedora |
  |--|--|--|--|--|--|
  | **ARM64** | :heavy_check_mark: | :heavy_check_mark: |  | :heavy_check_mark: | :heavy_check_mark: |
  | **PPC64le** | :heavy_check_mark: | :heavy_check_mark: |  |  | :heavy_check_mark: |
  | **x86_64** | :heavy_check_mark: |:heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: |




问题记录

```commandline
/usr/bin/docker-current: Error response from daemon: shim error: docker-runc not installed on system.
[root@bm48 ~]# locate docker-runc
/usr/libexec/docker/docker-runc-current
[root@bm48 ~]# ln -s /usr/libexec/docker/docker-runc-current /usr/libexec/docker/docker-runc

```

## CRI-O安装

匹配k8s版本，选择1.10,安装依赖包
```
yum install -y \
  btrfs-progs-devel \
  device-mapper-devel \
  git \
  glib2-devel \
  glibc-devel \
  glibc-static \
  go \
  golang-github-cpuguy83-go-md2man \
  gpgme-devel \
  libassuan-devel \
  libgpg-error-devel \
  libseccomp-devel \
  libselinux-devel \
  ostree-devel \
  pkgconfig \
  runc \
  skopeo-containers
```
现在源码切换到版本分支，并编译安装

```commandline
git clone https://github.com/kubernetes-incubator/cri-o 
git checkout -b release-1.10 remotes/origin/release-1.10
make install.tools
make BUILDTAGS=""
make install
make install.config
```

cni 网络配置
```commandline
go get -u -d github.com/containernetworking/plugins
cd plugins
./build/sh
mkdir -p /opt/cni/bin
cp bin/* /opt/cni/bin

## 添加网络配置文件
mkdir /etc/cni/net.d
cp $GOPATH/src/github.com/kubernetes-incubator/cri-o/contrib/* /etc/cni/net.d

## 创建cni0 bridge
brctl addbr cni0
```

修改`/etc/crio/crio.conf`
```commandline
[crio.runtime]
manage_network_ns_lifecycle = true
runtime = "/usr/bin/runc"
runtime_untrusted_workload = "/usr/bin/kata-runtime"
default_workload_trust = "untrusted"
```

启动cri-o

```commandline
make install.systemd
systemctl start crio
```

修改k8s环境变量，并重启k8s cluster

```commandline
CGROUP_DRIVER=systemd \
CONTAINER_RUNTIME=remote \
CONTAINER_RUNTIME_ENDPOINT='unix:///var/run/crio/crio.sock  --runtime-request-timeout=15m' \
./hack/local-up-cluster.sh
```
查看k8s服务状态

```commandline
 cluster/kubectl.sh get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}

```

创建测试pods
```commandline
cat >ngnix_untrusted.yam <<EON

apiVersion: v1
kind: Pod
metadata:
  name: nginx-untrusted
  annotations:
    io.kubernetes.cri.untrusted-workload: "true"
spec:
  containers:
    - name: nginx
      image: nginx
EON

cluster/kubectl.sh apply -f nginx-untrusted.yaml
```

查看pod

```commandline
cluster/kubectl.sh describe pod nginx-untrusted
```
根据pod输出的ip地址，可以正确访问ngnix的服务
```commandline
curl -i -XGET http://192.168.223.97:80
```

````text
# kata-runtime list
ID                                                                 PID         STATUS      BUNDLE                                                                                                                 CREATED                          OWNER
edff5f14efc36145ef29853064fafbb2d1e60c7127e5e62160457d7ebf362a6b   38346       running     /run/containers/storage/overlay-containers/edff5f14efc36145ef29853064fafbb2d1e60c7127e5e62160457d7ebf362a6b/userdata   2018-07-24T08:32:40.246491549Z   #0
0b530a50d5a353ef9f61e18b16c412fadf0fe98f82766f9f9678add7ede49d25   38543       running     /run/containers/storage/overlay-containers/0b530a50d5a353ef9f61e18b16c412fadf0fe98f82766f9f9678add7ede49d25/userdata   2018-07-24T08:33:21.832878116Z   #0



# df 
overlay                 241963588 33358740 208604848  14% /var/lib/containers/storage/overlay/1953579948b2a51cc93709ee96770496599e54f5f2cde525cc7138861a294495/merged
overlay                 241963588 33358740 208604848  14% /var/lib/containers/storage/overlay/93e320ca6218832f69984481a9ae945a2508b73ed4e3a17a69d0ee2a1aa54564/merged

````



# runc 

runc是docker贡献出来支持OCI的容器运行时项目，其实际上是在libcontainerd上封装了一层用于支持OCI，并提供CLI可以通过
[runtime spec](https://github.com/opencontainers/runtime-spec)运行容器。


## docker 问题

如果系统上安装了oci-register-machine，需要设置oci-register-machine为disable，否则kata container 退出时，
systemd-machined导致docker服务shutdown

```text
Jul 19 03:40:54 bm48 oci-register-machine[184480]: 2018/07/19 03:40:54 Register machine: poststop fa9f842ed3407606d01b7bb490473455be6bda49f6d478699976c61e83f37028 184468
Jul 19 03:40:54 bm48 systemd-machined[131968]: Machine fa9f842ed3407606d01b7bb490473455 terminated.
Jul 19 03:40:54 bm48 dockerd-current[183954]: time="2018-07-19T03:40:54.397828183-04:00" level=info msg="Processing signal 'terminated'"
Jul 19 03:40:54 bm48 dockerd-current[183954]: time="2018-07-19T03:40:54.397987798-04:00" level=debug msg="starting clean shutdown of all containers..."
```
当oci-register-machine设置为enable时，使用docker创建kata container，会在/var/run/systemd/machine下注册一条machine记录，而这条记录
关联的unit时docker服务，原因有待分析

```text
# ls -al /var/run/systemd/machines/
total 24
drwxr-xr-x.  2 root root 280 Jul 19 04:25 .
drwxr-xr-x. 18 root root 440 Jul 19 03:39 ..
-rw-r--r--.  1 root root 229 Jul 19 04:25 e04f4fe715665c32998b13250dab4b4a
lrwxrwxrwx.  1 root root  32 Jul 19 04:25 unit:docker.service -> e04f4fe715665c32998b13250dab4b4a
```

```text
/etc/oci-register-machine.conf
# Disable oci-register-machine by setting the disabled field to true
disabled : true
```

详细见[redhat bug#1514511](https://bugzilla.redhat.com/show_bug.cgi?id=1514511)


## kata container guest

kata container guest vm

application挂载实现
```text
-chardev socket,id=charch0,path=/run/vc/sbs/2ed4a3afed3c3d3269ca230d87da940bcdb85a6f239fab015b2710b83253dc02/kata.sock,server,nowait 
```

```text
 -device virtio-9p-pci,fsdev=extra-9p-kataShared,mount_tag=kataShared -fsdev local,id=extra-9p-kataShared,path=/run/kata-containers/shared/sandboxes/2ed4a3afed3c3d3269ca230d87da940bcdb85a6f239fab015b2710b83253dc02,security_model=none 
```

qemu nvdimm
```text
 -machine pc,nvdimm
 -m $RAM_SIZE,slots=$N,maxmem=$MAX_SIZE
 -object memory-backend-file,id=mem1,share=on,mem-path=$PATH,size=$NVDIMM_SIZE
 -device nvdimm,id=nvdimm1,memdev=mem1
```

```text
 -machine pc,accel=kvm,kernel_irqchip,nvdimm 
 -m 2048M,slots=2,maxmem=129554M 
 -device nvdimm,id=nv0,memdev=mem0
 -object memory-backend-file,id=mem0,mem-path=/usr/share/kata-containers/kata-containers.img,size=536870912
 -append tsc=reliable no_timer_check rcupdate.rcu_expedited=1 i8042.direct=1 i8042.dumbkbd=1 i8042.nopnp=1 \
    i8042.noaux=1 noreplace-smp reboot=k console=hvc0 console=hvc1 iommu=off cryptomgr.notests net.ifnames=0 \
    pci=lastbus=0 root=/dev/pmem0p1 rootflags=dax,data=ordered,errors=remount-ro rw rootfstype=ext4 debug \
    systemd.show_status=true systemd.log_level=debug panic=1 initcall_debug nr_cpus=48 \
    ip=::::::70694528ccaafd1e6c0cc593ae05a44536497c7aa381974566b49937e41dae39::off:: init=/usr/lib/systemd/systemd \
    systemd.unit=kata-containers.target systemd.mask=systemd-networkd.service systemd.mask=systemd-networkd.socket \
    agent.log=debug agent.sandbox=70694528ccaafd1e6c0cc593ae05a44536497c7aa381974566b49937e41dae39
```

![](https://github.com/kata-containers/documentation/blob/master/arch-images/DAX.png)


## kubevirt安装

virt-launcher调用libvirt创建虚拟机，即每个虚拟机会对应对立的libvirtd

虚拟机的root disk 生成方式两种：
1. registryDisk 有virt-handler将下载的image转为raw格式，然后启动virt-launcher将image作为root disk
2. 使用pvc(persistentvolumeclaim)，要求存在`/disk.img`文件

代码分析可见`virt-launcher/virtwrap/converter.go` volume支持的disk type均为file类型

不支持network block， 例如qemu/kvm rbd

思考：使用rbd的方式两种
1. 支持network block社区issue讨论
2. rbd map 使用qemu/kvm disk type=block

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

## virtlet 
### installation
```commandline
go get -u -d github.com/Mirantis/criproxy github.com/Mirantis/virtlet

bash -x build/cmd.sh build

kubectl label node kubdev --overwrite extraRuntime=virtlet
kubectl create configmap -n kube-system virtlet-image-translations --from-file  `pwd`/deploy/images.yaml


docker exec virtlet-build "${remote_project_dir}/_output/virtletctl" gen "--dev" >virtlet.yaml

```



### 问题记录

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
