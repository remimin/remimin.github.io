# Qemu架构分析

```commandline
/vl.c: 最主要的模拟循环，虚拟机机器环境初始化，和CPU的执行。
/target-arch/translate.c: 将客户机代码转化成不同架构的TCG操作码。
/tcg/tcg.c: 主要的TCG代码。
/tcg/arch/tcg-target.c: 将TCG代码转化生成主机代码
/cpu-exec.c: 其中的cpu-exec()函数主要寻找下一个TB（翻译代码块），如果没找到就请求得到下一个TB，并且操作生成的代码块。
```


# Qemu调试虚拟机方法

## Qemu monitor

Qemu创建虚拟机实例，可以通过```-mon```指定设备作为monitor，例如

```commandline

```



## Qemu Tracing

Qemu Tracing机制对于debug调试虚拟机来讲是比较有用的机制。


=======


## libvirt 日志
Libvirt日志也是非常有用的工具，需要根据自身的需要设置日志输出，否则很难找到需要的信息。同时输出的日志也能够了解到libvirt相关操作调用等信息。

Libvirt日志分级 DEBUG = 1, INFO = 2, WARNING = 3, ERROR = 4
支持log filter，格式是```x:name```。
x代表日志文件，name就是日志匹配的类型。例如 ```1:qemu``` 就代表qemu输出DEBUG日志。
对于常用的virsh命令，也可以通过环境变量的方式打印日志。
例如：

```LIBVIRT_DEBUG=error LIBVIRT_LOG_FILTERS="1:libvirt 1:qemu"```

表示日志默认输出为error, libvirt 和 qemu 日志级别为debug
如图所示：


![](/img/2018-05-10-qemu-tracing.png)

[https://libvirt.org/guide/html/Application_Development_Guide-Connections-Debug.html]
=======
![](/img/2018-05-10-qemu-tracing/virt-debug.png)



## KSM Tuning

/etc/ksmtuned.conf

## 查看硬件虚拟化

Intel CPU硬件虚拟化技术VT(Virtualization Technology) AMD的硬件虚拟化技术AMD-V(AMD Virtualization)。
需要再BIOS中设置开启硬件虚拟化。

Linux操作系统物理可以通过查看cpu info的flag来查询是否开启了硬件虚拟化：
- Intel CPU的 VT-x 会有vmx flag
- AMD CPU会有svm flag

```commandline
cat /proc/cpuinfo |egrep "vmx|svm"
```

> VMware ESXi可以通过配置开启硬件辅助使得vmware虚拟机可以使用宿主机的硬件虚拟化能力
> 
> ![](/img/2018-05-10-qemu-tracing/vmware_hardware_virt.png)


参考链接

  - https://libvirt.org/guide/html/Application_Development_Guide-Connections-Debug.html
  
>>>>>>> Stashed changes
