---
layout:     post
title:      "Qemu虚拟化之Machine Type"
subtitle:   ""
date:       2019-07-09
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - Qemu
    - Virtualization
---

# Qemu X86架构的Machine Type

* [Q35 vs. I440FX](#Q35-vs.-I440FX)
    * [i440fx/PIIX架构](#i440fx/PIIX架构)
    * [Q35架构](#Q35架构)
* [PCI vs. PCIe](#PCI-vs.-PCIe)
    * [PCI(Peripheral Component Interconnect)](#PCI(Peripheral Component Interconnect))
    * [PCIe(Peripheral Component Interconnect Express)](#PCIe(Peripheral Component Interconnect Express))


QEMU支持的X86架构非常少，在Q35出现之前，就只有诞生于1996年的i440FX + PIIX一个架构。一方面是Intel不断推出新的芯片组，
加入了PCIe、AHCI等等。i440FX已经无法满足需求，为此在 KVM Forum 2012 上Jason Baron带来了
[A New Chipset For Qemu - Intel's Q35](https://www.linux-kvm.org/images/0/06/2012-forum-Q35.pdf)。

## Q35 vs. I440FX

Q35是Intel在2007年9月推出的芯片组，与I440FX/PIIX差别在于：

* Q35 has IOMMU
* Q35 has PCIe
* Q35 has Super I/O chip with LPC interconnect
* Q35 has 12 USB ports
* Q35 SATA vs. PATA

Irq routing 

* Q35 PIRQ has 8 pins - PIRQ A-H
* Q35 has two PIC modes – legacy PIC vs I/O APIC
* Q35 runs in I/O APIC mode 
* Slots 0-24 are mapped to PIRQ E-H round robin
* PCIe Bus to PIRQ mappings can be programmed
    * Slots 25-31
* Q35 has 8 PCI IRQ vectors available, I440FX/PIIX4 only 2

I440FX/PIIX4 vs. Q35 devices

* AHCI vs. Legacy IDE
* PCI addresses
* Populate slots using flags
* Default slots

### i440fx/PIIX架构

Intel 440FX PMC（PCI and Memory Controller）为北桥芯片用于连接主板上的高速设备，向上提供了连接处理器的Host总线接口，
可以连接多个处理器，向下则主要提供了连接内存DRAM的接口和连接PCI总线系统的PCI总线接口，
通过该PCI root port扩展出整个PCI设备树，包括PIIX南桥芯片。

PIIX（PCI ISA IDE Xcelerator）南桥芯片则用于连接主板上的低速设备，主要包括IDE控制器、DMA控制器，USB控制器，
SMBus总线控制器，X-Bus控制器，USB控制，PIT（Programmable Interval Timer）， RTC（Real Time Clock，实时时钟)，
PIC（可编程中断控制器）等，并且提供PCI-ISA bridge链接ISA总线用于连接更多的低速设备，如键盘、鼠标、BIOS ROM等。

![I440FX](/img/2019-07-02-qemu_machine_type/I440FX.png)

## Q35架构

北桥为MCH(Memory Controller Hub )，南桥为ICH9(I/O Controller Hub)。CPU 通过前端总线(FSB)连接到北桥(MCH)，MCH链接内存，显卡，
高速PCIe接口等，南桥芯片则为USB，低速PCIe / SATA 等提供接入。

![Q35](/img/2019-07-02-qemu_machine_type/Q35.png)

![Motherboard_diagram](/img/2019-07-02-qemu_machine_type/Motherboard_diagram.png)



## PCI vs. PCIe

Q35与i440FX其中一个最大的差异就是支持PCIe，那么先了解一下PCI与PCIe的差别。

### PCI(Peripheral Component Interconnect)

PCI（Peripheral Component Interconnect）顾名思义就是外部设备的互联总线。PCI连接基于总线控制，
所有设备**共享双向并行总线**，
总线仲裁规则用于决定PCI挂载的外设什么时间可以访问总线。PCI总线最多支持32个外设，
可以通过PCI-to-PCI bridge(PPB)扩展PCI外设支持，如[图1](#PCI拓扑)，
Bus1/Bus2都是由Bus0通过PPB扩展的PCI总线。

![PCI拓扑](/img/2019-07-02-qemu_machine_type/PCI_topo.png)

PCI设备与CPU和内存通过主桥（host bridge）进行通信，如图[PCI体系结构](#PCI_compute_topo)，这种体系结构分隔了I/O总线与处理器的主机总线。

PCI主桥 (host bridge) 提供了CPU， 内存和外围组件之间的互连，CPU可以通过PCI主桥 (host bridge)直接访问独立于其他PCI总线主控器的主内存，
PCI设备也可以通过该主桥直接访问系统内存。

PCI主桥 (host bridge) 还可提供CPU和外围I/O设备之间的数据**地址映射**。该桥会将每个外围设备映射到主机地址域，
以便处理器可以通过地址域访问此设备。主桥(host bridge)也会将系统内存映射到PCI地址域，
以便PCI设备可以作为总线主控器访问主机内存。

![PCI_compute_topo](/img/2019-07-02-qemu_machine_type/PCI_compute_topo.gif)

X86架构采用了独立的地址空间，x86使用16位地址总线(也就是64K i/o地址空间)对io设备进行寻址，这个地址也称为i/o端口。在Linux操作系统中，可以通过`/proc/ioports`查看
系统的io端口。PCI包含三种不同的地址空间：配置、内存以及 I/O 空间。通过`/proc/ioports`可以看到`0cf8-0cff : PCI conf1`为PCI设备的
io端口。这8个字节实际上构成两个32位的寄存器，第一个是"地址寄存器"0cf8，第二个是"数据寄存器"0cff，CPU与PCI之间交互就是地址寄存器用于
配置和控制外设，数据寄存器用于数据传输。

![PCI conf space](/img/2019-07-02-qemu_machine_type/PCI_conf_space.png)


####　**配置空间**（configuration space）
   
    PCI配置空间为256B，是由PCI spec定义的，如图所示，配置空间对pci device是必须的。
    PCI头64B是固定的PCI configuration header由两种类型header Type 0代表是PCI设备，Type 1表明是PCI桥。
    
    **Device ID 和 Vendor ID**设备信息和厂商信息
    
    **Command寄存器** 
    存放了设备的配置信息，比如是否允许 Memory/IO 方式的总线操作、是否为主设备等。
    
   ![pci_command_reg](/img/2019-07-02-qemu_machine_type/pci_command_reg.png)
    
    **Status 寄存器**存放了设备的状态信息，比如中断状态、错误状态等。
   ![pci_status_reg](/img/2019-07-02-qemu_machine_type/pci_status_reg.png)
   
    **Header Type 寄存器**定义了设备类型，比如 PCI 设备、PCI 桥。

    **基址寄存器(BAR)**PCI设备的io地址空间和内存空间的基址，是可选的，当pci device需要配置时
    进行配置。基址寄存器的位 0 包含的值用于标识类型。位 0 中的值为 0 表示内存空间，值为 1 表示 I/O 空间。
    下图显示了两种基址寄存器： 一种表示内存类型，另一种表示 I/O 类型。PCI 设备最多有6个基址寄存器，
    而 PCI 桥最多有2个基址寄存器。
    
   ![pci_bar_mem](/img/2019-07-02-qemu_machine_type/bar_mem.png)
   
   ![pc_bar_mem](/img/2019-07-02-qemu_machine_type/bar_io.png)
   
    **Subordinate Bus Number，Secondary Bus Number 和 Primary Bus Number 寄存器**
   
    这三个寄存器只在 PCI 桥配置空间中存在，因为 PCI 桥会连接两条 PCI 总线，上行的总线被称为 Primary Bus，
    下行的总线被称为 Secondary Bus，Primary Bus Number 和 Secondary Bus Number 寄存器分别存储了上行和下行总线的编号，
    而 Subordinate Bus Number 寄存器则是存储了当前桥所能直接或者间接访问到的总线的最大编号。
    
    **Capabilities Pointer**
    
    开启Capbility支持需要在status register中，对应的bit设置为1. PCI设备capabilities是链表模式的。
    那么何谓Capabilities呢？它是PCI 2.2新加入的一个特性，之所以加入是因为当初规定所有PCI SPEC相关的配置寄存器都要放在PCI header内，
    到PCI2.2以后发现新加入的register在Configuration Header Space中放不下了，所以引入了Capabilities List。
    
* **PCI 内存地址空间(MMIO space)**
    
    对于不采用独立编址的系统架构，对于I/O设备的访问是通过内存映射的方式进行，MMIO space类似于
    X86系统的I/O port。

* **PCI I/O 地址空间(IO space)**

    PCI支持32位 I/O 空间。可以在不同的平台上以不同方式访问 I/O 空间。带有特殊 I/O 指令的处理器，如Intel处理器系列
    使用`in`和`out`指令访问I/O 空间。没有特殊I/O 指令的计算机将映射到对应于主机地址域中 PCI 主桥 (host bridge) 的地址位置。
    处理器访问内存映射的地址时，会向PCI主桥 (host bridge) 发送一个 I/O 请求，
    该主桥 (host bridge) 随后会将地址转换为 I/O 周期并将其放置在 PCI 总线上。内存映射的I/O通过处理器的本机装入/存储指令执行。
    
#### PCI 中断机制

在PCI协议中规定，每个设备支持对多四个中断信号，INTA，INTB，INTC，INTD。中断是电平触发的，当信号线为低电平，这引发中断。
一旦产生中断，则该信号线则一直保持低电平，直到驱动程序清除这个中断请求。这些信号线可以连接在一起。
在设备的配置空间里(Configuration Space)有一组寄存器(Interrupt Pin)表明一个功能(Function)使用哪个中断信号。

所有的中断信号都与系统的可编程中断路由器(Programmable Interrupt Router)相连，这个路由器有四个中断输入接口。
设备与这个路由器的中断输入接口怎么相连是系统的设计者在设计系统的时候就已经决定了，用户是不能改变的。


### PCIe(Peripheral Component Interconnect Express)

PCI Express (Peripheral Component Interconnect Express) 很明显就是高速的外部设备互联。
PCIe相对于PCI共享总线模式，采用的是Point-to-Point总线拓扑，PCIe中每个device都有自己的专属总线称为`link`，
`lane`是一对差分信号组成，分别负责收发数据（Rx/Tx)，也称作x1通道。
device之间的通路称为`link`，是由一组`lane`组成的单双工通道，n组`lane`组成的link提高了传输带宽。
通常看到的，PCIe x4等类似的，表明了通道数量为4。

![PCIe_lane_link](/img/2019-07-02-qemu_machine_type/link_lane.png)

Root Complex链接CPU memory 和PCIe设备。最多支持256 bus， 每个bus最多支持32个PCIe设备，
每个设备最多8个functions，每个function对应4KB配置空间。PCIe从硬件层面看与PCI由很大不同，如图所示。

![PCIe拓扑](/img/2019-07-02-qemu_machine_type/PCIe_topo.png)

![PCI_vs_PCIe](/img/2019-07-02-qemu_machine_type/PCI_vs_PCIe.png)


#### **PCIe配置空间**

    从软件层面看，PCIe配置空间从258B增加到了4KB，为了保证与PCI协议的兼容性，PCIe即支持io port访问，也支持mmio访问，
    但是超过256B的配置空间访问则只能通过MMIO的方式。
    
    ![PCIe_config_space](/img/2019-07-02-qemu_machine_type/PCIe_conf_space.png)

    *  **PCIe Extended Capabilities**

    与PCI的capabilities类似采用链表结构，只是起始的为100H
    
    * **PCIe ARI Extended Capabilities**

    PCIe链路采用端到端的通信方式，每一个链路只能挂接一个设备，因此在多数情况下，使用3位描述Device Number是多余的，
    因此PCIe V2.1总线规范提出了ARI (Alternative Routing-ID Interpretation)格式，ARI格式将ID号分为两个字段，
    分别为Bus号和Function号，而不使用Device号，ARI格式如图所示。
    
    |---|---|
    |7：0|7：0|
    |---|---|
    |Bus No. | Function No. |
    |---|---|
    
    PCIe总线引入ARI格式的依据是在一个PCIe链路上仅可能存在一个PCIe设备，因而其Device号一定为0。
    一个PCIe设备最多可以支持**256**个Function，而传统的PCIe设备最多只能支持8个Function。
    
    * **SR-IOV Extended Capability**
    
    SR-IOV(single root I/O virtualization)
    
    PCI-SIG定义的PCIe设备的I/O虚拟化标准，主要目标就是将单个的PCIe设备可以虚拟化为多个，包含两个概念PF & VF
    
    - PF (Physical Function)： 包含完整的PCIe功能，包括SR-IOV的扩张能力，该功能用于SR-IOV的配置和管理。
    - VF (Virtual Function)： 包含轻量级的PCIe功能。每一个VF有它自己独享的PCI配置区域，并且可能与其他VF共享着同一个物理资源
    
    * **其他的capabilities**
    
    - Function Level Reset (FLR)
    - Advanced Error Reporting (AER)
    - Access Control Services (ACS)
    - etc...

#### PCIe中断机制

    在PCI总线中，所有需要提交中断请求的设备，必须能够通过INTx引脚提交中断请求，而MSI机制是一个可选机制。而在PCIe总线中，
    PCIe设备必须支持MSI或者MSI-X中断请求机制，而可以不支持INTx中断消息。PCIe设备在提交MSI中断请求时，
    是向MSI/MSI-X Capability结构中的Message Address的地址写Message Data数据，从而组成一个存储器写TLP，向处理器提交中断请求。
    
    MSI和MSI-X机制的基本原理相同，其中MSI中断机制最多只能支持32个中断请求，而且要求中断向量连续，而MSI-X中断机制可以支持更多的中断请求，
    而并不要求中断向量连续。与MSI中断机制相比，MSI-X中断机制更为合理。
    

**参考链接**
    
1. [浅谈 Linux 内核开发之 PCI 设备驱动](https://www.ibm.com/developerworks/cn/linux/l-cn-pci/index.html)
2. [A New Chipset For Qemu - Intel's Q35](https://www.linux-kvm.org/images/0/06/2012-forum-Q35.pdf)
3. [PCI设备地址空间](https://www.cnblogs.com/zszmhd/archive/2012/05/08/2490105.html)
4. [详解io端口与io内存](http://mcu.eetrend.com/content/2018/100012210.html)
5. [PCI Express® Basics &Background](https://pcisig.com/sites/default/files/files/PCI_Express_Basics_Background.pdf)
6. [sr-iov](https://github.com/feiskyer/sdn-handbook/blob/master/linux/sr-iov.md)
7. [Linux设备驱动之PCI设备](http://www.deansys.com/doc/ldd3/ch12.html)
8. [PCI vs. PCIe](http://wiki.qemu.org/images/f/f6/PCIvsPCIe.pdf)
9. [PCI/PCIE 总线概述(5)—PCIE总线事务层](https://example61560.wordpress.com/2016/06/30/pcipcie-%e6%80%bb%e7%ba%bf%e6%a6%82%e8%bf%b05/)
10. [知乎PCIe专栏](https://zhuanlan.zhihu.com/PCI-Express)
11. [MSI和MSI-X中断机制](https://example61560.wordpress.com/2016/06/30/pcipcie-%E6%80%BB%E7%BA%BF%E6%A6%82%E8%BF%B06/)
12. [**Linux中断机制：硬件处理，初始化和中断处理**](http://www.10tiao.com/html/606/201808/2664605565/1.html)