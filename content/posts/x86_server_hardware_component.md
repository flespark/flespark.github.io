---
title: "服务器硬件组成和软件控制方法"
date: 2022-10-07T20:11:02+08:00
draft: false
categories: ["development"]
tags: ["x86", "server","linux"]
---



### 服务器硬件开发的特点

相对于PC机器，服务器要求更充分的性能释放和长时间稳定运行，除了投入成本选择更好的硬件，软件上完善的驱动和基于循证的配置调优更能取得事半功倍的效果。好比我们在组装电脑的时候，都会优先选择华硕，微星和技嘉的主板，因为这三家主板BIOS的可配置项目非常多，也提供许多辅助超频的功能。x86服务器硬件开发的目标与此类似，除了不同硬件模块（CPU，内存和网卡）的匹配以保证充分协调的性能释放，硬件选型和电路设计上也要提供一定的扩展性（网卡，显卡，加密卡，特殊加速卡的支持），较高的稳定行（电源电压和温度，风扇转速的监控，网口bypass和设备指示灯的控制）以及良好的互操作性能（远程网页管理，BMC以及其他管理接口的支持）。相比嵌入式开发，一方面服务器功能对于硬件的依赖程度相对较低。一是服务器各种行为只会用到CPU的通用计算能力，而嵌入式设备通常要同环境交互，各类多媒体数据编解码和信号处理的功能，以及低功耗和无线通信的要求，所以CPU演变成了SoC，二是桌面处理器领域一直以来都是intel一家独大的局面，所有围绕桌面CPU的外围硬件也都只能按照[intel的规矩][4]来，使用相同的通信和控制方法。三是服务器的硬件功能的迭代要比嵌入式设备慢得多，每一代产品的变化主要是CPU性能和总线吞吐的提升，同步代产品的多个规格又仅在性能高低上有所区别。另一方面，服务器CPU为了贴近日益增长的高性能，高带宽，低延迟和高度软件兼容的市场需求，所演进的技术方案（NUMA，VT-x，SRIOV），也给软件设计带来新的机遇和挑战。总的而言，嵌入式基本是面向不同工作场景的功能定制需求，基于intel生态的服务器开发具有面向统一协议和接口，面向高并发以及虚拟化的特点。

### x86服务器硬件组成

> 这里虽限定为x86服务器，但是依托国内的信创政策导向， 许多国产的非x86架构服务器（华为鲲鹏，飞腾，龙芯）也占据了一部分的市场。为了兼容现有的x86软件生态，硬件组成上非常相似

和大多数嵌入式数码产品一样，虽然服务器也在向着高度集成化的方向发展，但始终保持一定的扩展性。最基本的，至少具备扩展内存和外存的能力。我们都知道内存是直接与CPU通信的，但是各种外存使用的不同的通信协议（SATA，NVME， SAS），他们是怎样和CPU通信的呢。按照intel制定的规矩，x86上所有的高速硬件控制器都应该通过PCI接口与CPU通信，以获取最大的带宽和控制到最小的延迟。所以几乎所有的服务器都提供了PCI接口，用于连接网卡，外存控制器，显卡或者其他加速卡。对于较为高端的服务器，从CPU无法引出足够的PCI接口，就要借助芯片组（业内叫法intel称之为PCH，Platform Controller Hub，AMD称为FCH，Fusion Controller Hub，是主板上一块集成了众多控制器的面积较大的芯片），将CPU提供的DMI接口拓展为多个PCI接口，并提供额外的SATA，USB，网口接口，以及其他低速接口（串口，SPI，I2C等）和系统管理相关的功能。在组装台式家用电脑时，会有价格相差极大的许多型号的主板可供选择，例如支持intel 12代酷睿芯片的常见主板型号就有H610，B660， Z690，主板型号就指示主板上面的PC用芯片组型号。服务器的平台型号也指示了主板所使用的芯片组型号，例如常见的C3000，P620，P5000平台。芯片组只是扩展CPU接口的作用，对于低端服务器CPU提供的引脚就足够用了就不是必须的，但是一些系统必须的监控和管理功能，CPU无法提供，就要用到SuperIO芯片。即便是相对简单SuperIO芯片上也集成很多低速控制器。相对嵌入式产品而言服务器对于硬件成本是不敏感的，芯片提供的许多功能可能硬件电路上就没有引出或者软件上也没有实现。另外很多中高端服务器面板都可以看到一个类似于网口的IPMI接口，IPMI是独立于操作系统或者说CPU自行运作的管理接口，直接连接到单独的BMC芯片，BMC芯片和SuperIO功能有许多重合部分，但是多了一个以太网控制器以提供IPMI互联的能力，方便服务器集群的管理。总的来说，因为服务器上单个芯片的集成度更高，所以硬件组成上更为简单。

### x86硬件驱动的实现

提到x86硬件的驱动开发，你可能会关联到BIOS开发，好像操作系统层的驱动需要做的开发工作很少。这是因为x86过程中有一个实模式的阶段，BIOS一般都工作在实模式，按照intel的约定，BIOS中需要配置基本的驱动功能，然后把硬件信息按照ACPI和SMBIOS规范以特定格式保存在内存中的固定位置，操作系统启动后可以从这些位置获取到硬件的基本信息，或者根据这些信息对硬件做进一步的配置。其实现是，在操作系统层面，驱动从ACPI获取硬件寄存器在低端内存中的映射的位置，然后根据硬件的datasheet操作到想要的寄存器。前面提到目前阶段x86架构的所有硬件都是一级一级地挂在在PCI总线上，所以操作系统驱动也可以不使用ACPI提供的信息，直接从PCI配置空间获取硬件寄存器在低端内存中的映射。以下驱动配置的介绍均基于这种控制方法。

### PCI

PCI总线提供了外围设备与CPU高速通信的能力，是目前的intel的标准总线规范，PCIE是PCI的升级版，物理通信方式从并行改为了串行，增强了中断功能。PCI早已淘汰，目前主板能看到的都是PCIE插槽，这里只涉及PCI和PCIE通用的基本软件控制方式，所以以下不加区分。CPU通过总线号(8bit数据表示)，设备号(5bit)，功能号（3bit）[bus:dev.func]定位每一个PCI设备的功能单元（一个功能单元代表一个最小可独立配置硬件，例如一个4口网卡，每个网口分配一个功能号，设备号是一样的），在Linux上执行lspci列出系统识别到的每一个PCI设备，可以看到大多数PCI设备的总线号都为0，其他不为0的总线通过PCI桥的桥接的设备的总线号根据[深度优先遍历的顺序枚举][3]，执行lspci -tv可以列出PCI设备树，下面是在我使用的一款x86软路由上执行的结果。

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

设备树中4个网口（Ethernet Controller I225-V）分别通过4个PCI桥连接到CPU，即HOST bridge。PCI桥的总线地址是确认的，那么如何确认其所连接次级总线的PCI总线号呢，这里先要介绍PCI的配置空间。每个PCI设备都有256字节的配置空间，以每个寄存器32bits大小的形式对齐访问。其中前4个寄存器有着相同的标识功能，后面寄存器的数据区分设备类型表示不同的[含义][1]。Linux系统访问PCI设备的配置空间有2种方式。

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

或使用lspci或者[setpci][8],[pci_debug][9]命令读写PCI配置空间和内存映射寄存器的内容。

读写PCI的配置空间首先需要确定设备的PCI地址，但是接在PCI桥上设备的PCI的总线地址不是固定的。常见的PCI桥有2中，一种Host Bridge，一种PCI-to-PCI Bridge，Host Bridge的插槽上的每一个设备的总线号对应Host Bridge一个新的功能号，PCI-to-PCI Bridge的功能号个数是固定的，读取PCI配置空间的Class code可以区分Bridge类型。对于Bridge类型的PCI设备配置空间的Subordinate bus number则保存着次级设备的总线号，PCI插槽在接入新设备时，设备号和功能号都是固定的，通过读取该寄存器中的内容确认新设备的总线号确认新设备的PCI地址。

[PCIE][5]较之PCI地址中多了一个PCI域（segment group number）的部分，设备的配置空间升级到了4096字节的大小，前面的256字节和PCI一样可以通过上面PCI的传统方式访问，但是扩展的部分（包括PCI域不是默认0设备）的寄存器，x86设备通过BIOS传递的ACPI中的MCFG表获取配置空间在内核中的映射地址，其他架构系统则一般通过设备树的方式。

BIOS通过操作配置空间只完成的PCI层面的配置（包括PCI的设备ID，PCI地址，速率，中断号分配），况且系统启动之后这块内容被置为只读。功能层面（对应各种外围设备控制器，例如SMBus控制器，NvMe控制器，MAC芯片）则是通过对多个BAR（Base Address Register）映射位置的内存来完成配置，BAR可以在配置空间获取到，Endpoint设备的配置空间有5个BAR，Bridge设备配置空间只有2个BAR。Linux中内核驱动代码主要配置这部分内存中的数据，但是驱动加载成功之后，PCI与CPU的通信就是走的DMA通道。

PCI总线枚举和设备识别是在BIOS中进行的，主板BIOS配置页面可以找到PCI的速率，中断模式，电源管理的一些基础配置项，有的BIOS固定把部分PCI桥的次级设备设置的固定bus_num。进入Linux系统后，总是需要通过lspci读取大致的的PCI设备工作状态。

### SuperIO

[SuperIO][6]多在工控机和低端服务器以及一些x86嵌入式设备上比较常见，是x86在ISA总线架构留下的产物，理应该被淘汰了。而且SuperIO和现代桌面CPU所能提供的功能有一部分的重叠，但是很多低端机上还是有用到SuperIO，一方面作为替代PCH的低成本选项[扩展低速接口控制器和引脚][7]，另一方面在多种产品中使用相同的SuperIO也减少了硬件制版和软件适配上的开销。SuperIO提供的低速总线扩展，包括串口接入，SMBus/I2C接口，温度传感器，风扇转速控制，供电电压监控，看门狗中断，红外信号接收，PS/2键鼠接口等几乎所有的低速总线功能，不同型号所支持的功能和对应接口数量差异也比较大，具体以厂商提供的datasheet位置。以前SuperIO是通过ISA总线与CPU交互数据的，现在多数通过LPC接口与CPU进行通信，但是软件上还是沿用原来直接读写IO端口的控制方式。因为只有x86架构支持IO端口命令，所以几乎也只能在x86平台服务器中找到SuperIO芯片。通过2个映射到特定IO端口的寄存器，读写SuperIO每个对应功能的对应配置寄存器：

- 索引寄存器（大多数芯片映射到0x2E的位置，少部分映射到0x4E）

- 数据寄存器（大多数映射到0x2F位置，少部分映射到0x4F）
  

首先往索引寄存器端口写入想要读写寄存器的索引值，然后数据寄存器端口就可以读写到想要的数据。SuperIO读写一个配置寄存器往往需要经过多层索引，首先芯片的每一个功能模块（比如温度传感器1）占据一段IO端口地址段，他们的起始地址可以通过LDN（Logic Device Number）定位，流程如下：

```c
outb(0x20, index_reg);  //基偏移为0x20位置的寄存器，对应SuperIO的ID
int id = inb(data_reg); //保存ID，确认芯片型号识别正确

outb(0x07, index_reg);  //基偏移为0x07的寄存器，对应LDN选择寄存器
outb(0x03, data_reg);   //选择LDN 0x03

out(0x30， index_reg);  //选择LDN偏移为0x30的使能寄存器
outb(0x01, data_reg);   //使能对应的功能
```

有的SuperIO控制流程更复杂一些，在写入寄存器之前有固定的entry instruction，并且把多个功能单元合并到一个LDN访问，以下代码示例借助[ioport][10]控制ITE8772E SuperIO的PWM风扇1转速：

```bash
#!/bin/bash

set -e

# entry instru
it87_entry() {
outb 0x2e 0x87
outb 0x2e 0x01
outb 0x2e 0x55
outb 0x2e 0x55
}

# select LDN
EU_probe() {
# select EU(Environment Controller Unit)
outb 0x2e 0x07
outb 0x2f 0x04

# read EU activate register
outb 0x2e 0x30
tmp=$(inb 0x2f)

if [[ $? -ne 0 ]] || [[ $(( $tmp & 0x01 )) -ne 1 ]]; then
	printf "IT87 not activited!\n"
	exit
fi

# read EU base register address
outb 0x2e 0x60
base_addr=$(inb 0x2f)
base_addr=$(( base_addr << 8 ))
outb 0x2e 0x61
base_addr=$(( base_addr | $(inb 0x2f) ))
echo $base_addr
}

get_fan_ctl_reg() {
if (($# < 1)); then
	echo "not designated base addr in arg"
	exit 1
fi

base_addr=$1

# get EU address map in LPC bus
EU_addr=$(( base_addr + 0x05 ))
EU_data=$(( base_addr + 0x06 ))

if [[ -n $targetVal ]]; then
	# set fan1 software control mode
	outb $EU_addr 0x16
	outb $EU_data 0
	# set fan1 PWM duty value(per 256)
	outb $EU_addr 0x6b
	outb $EU_data $targetVal
fi
outb $EU_addr 0x6b
fanOutput1=$(inb $EU_data)
printf "fan1 output 0x%x\n" "$fanOutput1"
}

# exit instru
it87_exit() {
outb 0x2e 0x02
outb 0x2f 0x02
}

main() (
	it87_entry
	base_addr=`EU_probe`
if (($# == 1)); then
	targetVal=$(( $1 & 0xff ))
fi
	get_fan_ctl_reg $base_addr
	
	it87_exit
)

main $@
```

很多SuperIO可以找到对应的内核模块，加载后通过sysfs控制SuperIO寄存器的是一种更规范的行为，下面的守护进程每隔5秒读取一次SuperIO的温度传感器的数据调整风扇的转速：

```bash
#!/bin/bash

tweak_fan()
(
    while true; do
        sio_temp=$(($(cat temp2_input) / 1000))
        if ((sio_temp < 45)); then
            echo 32 > pwm2
        elif ((sio_temp < 55)); then
            echo 64 > pwm2
        elif ((sio_temp < 65)); then
            echo 128 > pwm2
        else
            echo 255 > pwm2
        fi
        sleep 5
    done
)

main()
(
    modprobe it87 || exit -1
    cd /sys/class/hwmon/hwmon2
    tweak_fan > /dev/null
)

main &
```

想要知道主板上是否有SuperIO芯片，除了打开机箱找到芯片，[lm-sensor][11]提供的sensors-detect工具多数情况下也可以正确识别。对于一些后期推出不太流行的SuperIO芯片，sensors-detect的识别结果就仅供参考，但BIOS的配置菜单不会骗人。虽然SuperIO提供的众多功能单元，但是硬件上引出有效连线不会很多，把SuperIO利用起来都要经过一些手动测试。在内核中加入一款SuperIO的支持并不困难，只要参考相似型号的已有代码，基于[hwmon][12]做一些配置修改的琐碎工作，所以这部分内核代码最近都没什么人改动。

### PCH

从英特尔推出酷睿处理器开始，内存控制器集成到了CPU内部，主板中就没有了北桥这一说，PCH就继承了南桥的功能。按以往南桥连接低速总线的说法也是相对的，PCH也提供了多路PCIe，千兆网卡的高速连接能力。intel PCH与CPU之间通信是通过一种类PCI的DMI高速总线，PCH支持的系统监控能力比SuperIO更加丰富，所以和CPU之间还有额外的许多中断线连接。受限于DMI总线的带宽，PCH必须在其提供的各种高速总线接入功能（PCIe，USB，GbE，SATA）中做出取舍，这部分控制结构称之为HSIO（Flexible IO）。反映到PC主板上，如果把所有的SATA接口插满的话，通过PCH桥接的PCIe口就没法用了。PCH所提供的低速接口和SuperIO区别不大，但是控制上多数是基于PCI总线，也就是需要从PCI配置空间获取BAR，然后才可以操作具体的寄存器，具体参考intel提供的datasheet。

### SMbus/I2C

SMBus是intel基于I2C协议新增网络层控制层而推出的一种总线协议，虽然在虽然在物理层面也有一些区别，但是SMBus是完全兼容I2C的软件框架的。SMBus一般只出现在intel x86的机器上（CPU，PCH或者SuperIO带有SMBus控制器），而国产化平台上则换成了I2C，无歧义的情况下这里统称I2C。I2C可以连接一些低速外围部件，例如EEPROM、GPIO扩展板、继电器，常留给下游厂商或者部门做一些产线的区分和个性化定制（俗话说科技以换壳为本）。I2C作为最常用的总线协议之一，内核是提供了标准的ioctl接口和调试工具i2c-tools，所以基于已有内核驱动控制I2C设备相对简单，intel x86平台的SMBus控制器也可以通过IO端口控制。在一些的系统上，每个I2C设备驱动加载的顺序不是固定的，导致/dev目录的命名可能在启动后会变动。但是控制器的编号是固定的。所以要通过sysfs定位定位到对应的控制器。在设备带有多个网卡插槽的情况下，I2C控制器通过一个I2C Mux连接到多张网卡的EEPROM。I2C Mux一般提供了控制地址和转发地址，读写定路由地址可以控制I2C Mux选路，读写路由地址则相当于直接选通的末端网卡通信。另外，由于一个I2C控制是可能连接到多个设备，或者作为接口提供服务的情况下，内部需要做好访问互斥控制，不然可能会导致一些难以排查的bug。

### 高并发与负载均衡

应用层面我们可以尽量减少上下文切换和使用细粒度锁以程度更高的并发，但是如果要充分发挥硬件的价值，还需要借助操作系统层面的一些接口以实现最大并发数和负载均衡。这里以Linux上网络流量数据面的应用为例，尝试说明这些接口的使用方法。随着芯片技术与高速网络接口技术的一日千里式发展，报文吞吐需要处理10Gbps端口处理能力，世面上大量的25G、 40G甚至100G高速端口已经出现，主流处理器的主频仍停留在3GHz左右。IO超越CPU的运行速率，是横在行业面前的技术挑战。intel推出了充分利用自家处理器功能的一套源码编程库dpdk，dpdk利用Linux提供的大页内存，CPU绑定和亲和性，轮询机制，可以显著提高基础数据平面功能。大页内存可以将系统中部分的物理内存从默认的4k大小块映射改为2M或者1G的容量，这样一个TLB表项以指向更大的内存区域，这样可以大幅减少 TLB miss 的发生。dpdk所有使用的内存都是从HugePage里分配，构建一个内存池并预先分配好同样大小的 mbuf，供每一个数据包使用。这部分预留HugePage的大小往往根据硬件大小不同需要进行一些调整，一般预留1G的基础内存给系统的其他服务，剩下的所有内存都以大页内存的形式留给dpdk。配置透明HugePage的方式为：

  1. 在Linux启动命令中配置预留大页内存的大小：
  2. 在文件系统配置文件添加大页内存的默认挂载节点
  3. 在dpdk配置文件中设置大页内存设备的路径

这样在dpdk应用中，预留的大页内存就可以直接以块设备文件形式直接访问了。

默认情况下，Linux中运行的所有进程随机的在所有core上面调度，但是数据面运行的进程大多使用相同的上下文内容，dataplane与其他系统服务进程之间的切换导致cache miss会带来总体的系统延迟。同时对于某些包含多个NUMA node的中高端处理器，不同的PCI总线直连到不同的NUME node，跨node的PCI设备访问比node内的要慢，所以指定process-core-port的一对一绑定关系尽可能的减少网络报文的访问延迟。

在较低的流量下，网卡中断可以即使响应报文的处理，但是流量一旦突破瓶颈，由于中断耗费了太多时间在内核的中断上下文切换的过程，新的报文得不到及时处理，后续的丢包会变得越来越严重。**TODO**

### Intel虚拟化技术

TODO

[1]: https://wiki.osdev.org/Acpi "PCI - OSDev Wiki"

[2]: https://www.kernel.org/doc/html/latest/PCI/sysfs-pci.html "Accessing PCI device resources through sysfs"

[3]: https://mp.weixin.qq.com/s/ulSSIRiOFGPeY6s7_kHuNg "一文梳理PCIe技术原理"

[4]: https://stackoverflow.com/questions/6852332/how-does-cpu-communicate-with-peripherals "How does cpu communicate with peripherals? - Stack Overflow"

[5]: https://wiki.osdev.org/PCI_Express#Enhanced_Configuration_Mechanism "PCI Express"

[6]: https://web.archive.org/web/20210415014544/https://www.coreboot.org/Developer_Manual/Super_IO "Developer Manual/Super IO - coreboot"

[7]: https://en.wikipedia.org/wiki/Super_I/O "Super I/O - Wikipedia"

[8]: https://www.man7.org/linux/man-pages/man8/setpci.8.html "setpci(8) -Linux manual page"

[9]: http://www.armadeus.org/wiki/index.php?title=Pci_debug "Pci debug -ArmadeusWiki"

[10]: https://people.redhat.com/rjones/ioport/ "ioport - direct access to I/O ports from the command line"

[11]: https://github.com/lm-sensors/lm-sensors "github: lm-sensors"

[12]: https://www.kernel.org/doc/html/latest/hwmon/hwmon-kernel-api.html "The Linux Hardware Monitoring Kernel API"
