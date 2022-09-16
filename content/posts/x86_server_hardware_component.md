---
title: "服务器硬件组成和软件控制方法"
date: 2022-07-30T20:11:02+08:00
draft: true
categories: ["development"]
tags: ["x86", "server","linux"]
---



### 服务器硬件开发的特点

任何服务器的高效稳定运行，都离不开对连接到CPU的外围硬件的有效管理。就好比我们组装电脑的时候，都会优先选择华硕，微星和技嘉的主板，因为他们的主板BIOS的可配置项目非常多，也提供许多辅助超频的功能。x86服务器硬件的功能也是这样，除了整体硬件（CPU，内存和网卡）的匹配保证CPU的性能的释放，也要保证设备有一定的扩展性（网卡，显卡，加密卡，特殊加速卡的支持），较高的稳定行（电源电压和温度，风扇转速的监控，网口bypass和设备指示灯的控制）以及良好的互操作性能（硬件上主要是BMC的支持）。相对于嵌入式开发而言，服务器功能对于硬件的依赖程度相对较低。一是服务器各种行为只会用到CPU的通用计算能力，而嵌入式设备通常要和同环境交互，应对各类媒体计算和信号处理的功能，通常还有低功耗和无线通信的要求，所以CPU演变成了SoC，二是桌面处理器领域一直以来都是intel一家独大的局面，所有围绕桌面CPU的外围硬件也都只能按照intel的规矩来，使用相同的通信和控制方法。三是服务器的硬件功能的迭代要比嵌入式设备慢得多，每一代产品的变化主要是CPU性能和总线吞吐的提升，同步代产品的多个规格又仅在性能高低上有所区别。所以相对于嵌入式面向不同工作场景的功能定制需求，基于intel生态的服务器开发具有面向统一协议和接口，面向高性能以及虚拟化的特点。

### x86服务器硬件组成

> 这里虽限定为x86服务器，但是依托国内的信创政策导向， 许多国产的非x86架构服务器（华为鲲鹏，飞腾，龙芯）也占据了一部分的市场。为了兼容现有的x86软件生态，硬件组成上非常相似

和大多数嵌入式数码产品一样，虽然服务器也在向着高度集成化的方向发展，但始终保持一定的扩展性。最基本的，至少具备扩展内存和外存的能力。我们都知道内存是直接与CPU通信的，但是各种外存使用的不同的通信协议（SATA，NVME， SAS），他们是怎样和CPU通信的呢。按照intel制定的规矩，x86上所有的高速硬件控制器都应该通过PCI接口与CPU通信，以获取最大的带宽和控制到最小的延迟。所以几乎所有的服务器都提供了PCI接口，用于连接网卡，外存控制器，显卡或者其他加速卡。对于较为高端的服务器，从CPU无法引出足够的PCI接口，就要借助芯片组（业内叫法intel称之为PCH，Platform Controller Hub，AMD称为FCH，Fusion Controller Hub，是主板上一块集成了众多控制器的面积较大的芯片），将CPU提供的DMI接口拓展为多个PCI接口，并提供额外的SATA，USB，网口接口，以及其他低速接口（串口，SPI，I2C等）和系统管理相关的功能。在组装台式家用电脑时，会有价格相差极大的许多型号的主板可供选择，例如支持intel 12代酷睿芯片的常见主板型号就有H610，B660， Z690，主板型号就指示主板上面的PC用芯片组型号。服务器的平台型号也指示了主板所使用的芯片组型号，例如常见的C3000，P620，P5000平台。芯片组只是扩展CPU接口的作用，对于低端服务器CPU提供的引脚就足够用了就不是必须的，但是一些系统必须的监控和管理功能，CPU无法提供，就要用到SuperIO芯片。即便是相对简单SuperIO芯片上也集成很多低速控制器。相对嵌入式产品而言服务器对于硬件成本是不敏感的，芯片提供的许多功能可能硬件电路上就没有引出或者软件上也没有实现。另外很多中高端服务器面板都可以看到一个类似于网口的IPMI接口，IPMI是独立于操作系统或者说CPU自行运作的管理接口，直接连接到单独的BMC芯片，BMC芯片和SuperIO功能有许多重合部分，但是多了一个以太网控制器以提供IPMI互联的能力，方便服务器集群的管理。总的来说，因为服务器上单个芯片的集成度更高，所以硬件组成上更为简单。

### x86硬件驱动的实现

提到x86硬件的驱动开发，你可能会关联到BIOS开发，好像操作系统层的驱动需要做的开发工作很少。这是因为x86过程中有一个实模式的阶段，BIOS一般都工作在实模式，按照intel的约定，BIOS中需要配置基本的驱动功能，然后把硬件信息按照ACPI和SMBIOS规范以特定格式保存在内存中的固定位置，操作系统启动后可以从这些位置获取到硬件的基本信息，或者根据这些信息对硬件做进一步的配置。其实现是，在操作系统层面，驱动从ACPI获取硬件寄存器在低端内存中的映射的位置，然后根据硬件的datasheet操作到想要的寄存器。前面提到目前阶段x86架构的所有硬件都是一级一级地挂在在PCI总线上，所以操作系统驱动也可以不使用ACPI提供的信息，直接从PCI配置空间获取硬件寄存器在低端内存中的映射。以下驱动配置的介绍均基于这种控制方法。

### PCI

PCI总线提供了外围设备与CPU高速通信的能力，是目前的intel的标准总线规范，PCIE是PCI的升级版，物理通信方式从并行改为了串行，增强了中断功能。PCI早已淘汰，目前主板能看到的都是PCIE插槽，PCI和PCIE的软件控制方式没有区别，一般驱动只涉及到PCI的基础功能，所以这里不加区分。CPU通过总线号，设备号，功能号[bus:dev.func]定位每一个PCI设备的功能单元（一个功能单元代表一个最小可独立配置硬件，例如一个4口网卡，每个网口分配一个功能号，设备号是一样的），在Linux上执行lspci列出系统识别到的每一个PCI设备，可以看到大多数PCI设备的总线号都为0，其他不为0的总线通过PCI桥的桥接的设备的总线号根据广度优先遍历的顺序枚举，执行lspci -tv可以列出PCI设备的拓扑树。
```sh
root@edge-xnet:~/Desktop# lspci
00:00.0 Host bridge: Intel Corporation Gemini Lake Host Bridge (rev 06)
00:02.0 VGA compatible controller: Intel Corporation GeminiLake [UHD Graphics 600] (rev 06)
00:0e.0 Audio device: Intel Corporation Celeron/Pentium Silver Processor High Definition Audio (rev 06)
00:0f.0 Communication controller: Intel Corporation Celeron/Pentium Silver Processor Trusted Execution Engine Interface (rev 06)
00:12.0 SATA controller: Intel Corporation Celeron/Pentium Silver Processor SATA Controller (rev 06)
00:13.0 PCI bridge: Intel Corporation Gemini Lake PCI Express Root Port (rev f6)
00:13.1 PCI bridge: Intel Corporation Gemini Lake PCI Express Root Port (rev f6)
00:13.2 PCI bridge: Intel Corporation Gemini Lake PCI Express Root Port (rev f6)
00:13.3 PCI bridge: Intel Corporation Gemini Lake PCI Express Root Port (rev f6)
00:14.0 PCI bridge: Intel Corporation Gemini Lake PCI Express Root Port (rev f6)
00:14.1 PCI bridge: Intel Corporation Gemini Lake PCI Express Root Port (rev f6)
00:15.0 USB controller: Intel Corporation Celeron/Pentium Silver Processor USB 3.0 xHCI Controller (rev 06)
00:1f.0 ISA bridge: Intel Corporation Celeron/Pentium Silver Processor LPC Controller (rev 06)
00:1f.1 SMBus: Intel Corporation Celeron/Pentium Silver Processor Gaussian Mixture Model (rev 06)
01:00.0 Ethernet controller: Intel Corporation Ethernet Controller I225-V (rev 03)
02:00.0 Ethernet controller: Intel Corporation Ethernet Controller I225-V (rev 03)
03:00.0 Ethernet controller: Intel Corporation Ethernet Controller I225-V (rev 03)
04:00.0 Ethernet controller: Intel Corporation Ethernet Controller I225-V (rev 03)
06:00.0 Network controller: Intel Corporation Device 2725 (rev 1a)
root@edge-xnet:~/Desktop# lspci -tv
-[0000:00]-+-00.0  Intel Corporation Gemini Lake Host Bridge
           +-02.0  Intel Corporation GeminiLake [UHD Graphics 600]
           +-0e.0  Intel Corporation Celeron/Pentium Silver Processor High Definition Audio
           +-0f.0  Intel Corporation Celeron/Pentium Silver Processor Trusted Execution Engine Interface
           +-12.0  Intel Corporation Celeron/Pentium Silver Processor SATA Controller
           +-13.0-[01]----00.0  Intel Corporation Ethernet Controller I225-V
           +-13.1-[02]----00.0  Intel Corporation Ethernet Controller I225-V
           +-13.2-[03]----00.0  Intel Corporation Ethernet Controller I225-V
           +-13.3-[04]----00.0  Intel Corporation Ethernet Controller I225-V
           +-14.0-[05]--
           +-14.1-[06]----00.0  Intel Corporation Device 2725
           +-15.0  Intel Corporation Celeron/Pentium Silver Processor USB 3.0 xHCI Controller
           +-1f.0  Intel Corporation Celeron/Pentium Silver Processor LPC Controller
           \-1f.1  Intel Corporation Celeron/Pentium Silver Processor Gaussian Mixture Model
```

拓扑树中4个网口（Ethernet Controller I225-V）分别通过4个PCI桥连接到CPU，即HOST bridge。PCI桥的总线地址是确认的，那么如何确认其所连接次级总线的PCI总线号呢，这里先要介绍PCI的配置空间。每个PCI设备都有256字节的配置空间，以每个寄存器32bits大小的形式对齐访问。其中前4个寄存器有着相同的标识功能，后面寄存器的数据区分设备类型表示不同的[含义][1]。Linux系统访问PCI设备的配置空间有2种方式。

第一种方式：直接从低端IO空间读写PCI寄存器的数据，参考代码：

```c
uint16_t pciConfigReadWord(uint8_t bus, uint8_t slot, uint8_t func, uint8_t offset) {
    uint32_t address;
    uint32_t lbus  = (uint32_t)bus;
    uint32_t lslot = (uint32_t)slot;
    uint32_t lfunc = (uint32_t)func;
    uint16_t tmp = 0;
 
    // Create configuration address as per Figure 1
    address = (uint32_t)((lbus << 16) | (lslot << 11) |
              (lfunc << 8) | (offset & 0xFC) | ((uint32_t)0x80000000));
 
    // Write out the address
    outl(0xCF8, address);
    // Read in the data
    // (offset & 2) * 8) = 0 will choose the first word of the 32-bit register
    tmp = (uint16_t)((inl(0xCFC) >> ((offset & 2) * 8)) & 0xFFFF);
    return tmp;
}
```

第二种方式：从[sysfs][2]中获取，参考代码：

```c
uint32_t pciConfigReadWord(uint8_t bus, uint8_t slot, uint8_t func, uint8_t offset) {
    char pci_path[PCI_PATH_LENGTH];
    int fd = -1;
    int i = 0, j = 0;
    uint32_t reg_val;
#if defined(__x86_64__)
    #define PCI_PATH_PATTERN "/sys/bus/pci/devices/0000:%02x:%02x.%1x/config"
#elif defined(__aarch64__)
    #define PCI_PATH_PATTERN "/sys/bus/pci/devices/pci0000:%02x:%02x.%1x/config"
#endif
    snprintf(pci_path, PCI_PATH_LENGTH - 1, PCI_PATH_PATTERN,
        bus, dev, func);
    fd = open(pci_path, O_RDWR | O_SYNC);
    if (fd < 0) {
        return -1;
    }
    lseek(fd, offset & 0xfc, SEEK_SET);
    read(fd, &reg_val, 4);
    close(fd);
    return reg_val;
}
```

或者使用lspci或者setpci,pci_debug命令读写PCI配置空间和内存映射寄存器的内容。

读写PCI的配置空间首先需要确定设备的PCI地址，但是接在PCI桥上设备的PCI的总线地址不是固定的。常见的PCI桥有2中，一种Host Bridge，一种PCI-to-PCI Bridge，Host Bridge的插槽上的每一个设备的总线号对应Host Bridge一个新的功能号，PCI-to-PCI Bridge的功能号个数是固定的，读取PCI配置空间的Class code可以区分Bridge类型。对于Bridge类型的PCI设备配置空间的Subordinate bus number则保存着次级设备的总线号，PCI插槽在接入新设备时，设备号和功能号都是固定的，通过读取该寄存器中的内容确认新设备的总线号确认新设备的PCI地址。




![How does cpu communicate with peripherals? - Stack Overflow] https://stackoverflow.com/questions/6852332/how-does-cpu-communicate-with-peripherals

[1]: https://wiki.osdev.org/Acpi "PCI - OSDev Wiki"

[2]: https://www.kernel.org/doc/html/latest/PCI/sysfs-pci.html "Accessing PCI device resources through sysfs"
