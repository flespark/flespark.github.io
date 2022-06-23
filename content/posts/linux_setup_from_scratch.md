---
title: "Linux_setup_from_scratch"
date: 2021-06-25T15:23:22+08:00
draft: tru
catagories: ["development"]
tags: ["config", "linux"]
---



> 接触Linux也有5年了，从事Linux嵌入式开发工作也有5年，Linux作为作为一款支持硬件平台最广泛的开源操作系统，现在还不知道怎么构建一个定制化的Linux那是实在有点说不过去了，所以就有了这篇Linux系统构建的思路。

一直以来手头使用我在开发过程中PC和服务器中使用的Linux发行版除了Ubuntu就是CentOS，产品开发也是完全基于芯片厂商提供的SDK。但是最近计划切换到Arch发行版，又系统地学习了一遍操作系统，认识了构建Linux系统镜像需要用到的各种工具。于是尝试手动配置和构建内核，以研究内核运行的机制。



###	构建运行在QEMU上的最小Linux系统

在我们一般的印象中，发展至今的Linux为了做到给 各种处理器架构和硬件的支持，已经集成了数千万行的代码。但是源码中大部分的驱动编译选项默认都是关闭的，所以编译出来的Linux内核镜像并没有多大，过往版本的Linux的代码量相对更小，如果我们用不上新版提供的功能和特性，选择较老版本编译出来的空间可以做到相当小。那么有多小呢，这里以Android10对应的4.10版本为例，其他版本的编译方式略有不同，有的可能无法直接编译。Linux各个版本的改动可以参考[LinuxNewbies][1]。

首先，从国内的镜像站下载对应版本的内核源码压缩包并解压

```bash
wget https://mirrors.cloud.tencent.com/linux-kernel/v4.x/linux-4.10.tar.xz
tar -xvf linux-4.10.tar.xz
cd linux-4.10
```

如果是二次编译先清理工程

```bash
make mrproper -j $(nproc)
```

读取分散于各个目录的配置，生成一个默认的配置文件，稍加配置，启动编译

```bash
# 没有指定ARCH参数的话会在根目录下生成名为.config的全局配置文件,没有指定ARCH参数则默认为x86_64平台
make defconfig -j $(nproc)
# 设置主机名字
sed -i "s/.*CONFIG_DEFAULT_HOSTNAME.*/CONFIG_DEFAULT_HOSTNAME=\"flespark\"/" .config
# 使用zlib的压缩格式为xz，生成文件体积最小
sed -i "s/.*\\(CONFIG_KERNEL_.*\\)=y/\\#\\ \\1 is not set/" .config
sed -i "s/.*CONFIG_KERNEL_XZ.*/CONFIG_KERNEL_XZ=y/" .config
# 指定一些gcc参数，生成内核文件中需要加载到内存的部分使用zlib压缩，启动编译
make CFLAGS="-Os -s -fno-stack-protector -fomit-frame-pointer -U_FORTIFY_SOURCE" bzImage -j $(nproc)
```

编译好的内核文件可以在arch/x86_64/boot目录中找到，接下来，还需要编译一个静态链接的busybox作为shell。

下载最新1.32.1版本的busybox，解压

```bash
wget http://busybox.net/downloads/busybox-1.32.1.tar.bz2 && tar xvf busybox-1.32.1.tar.bz2 && cd busybox-1.32.1
```

编译主机和目标机器都是x86_64架构，静态链接编译只需要修改这两项配置即可

```bash
sed -i "s/.*CONFIG_STATIC.*/CONFIG_STATIC=y/" .config
sed -i "s|.*CONFIG_EXTRA_CFLAGS.*|CONFIG_EXTRA_CFLAGS=\"-Os -s -fno-stack-protector -fomit-frame-pointer -U_FORTIFY_SOURCE\"|" .config
```

编译可能会以这样的报错结束`Static linking against glibc, can't use --gc-section`，不用管他，检查静态链接的busybox是否已经生成

```bash
$ file busybox
busybox: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=3f84fd0fa0f21c3354d3c6b52e013721c261918f, for GNU/Linux 3.2.0, stripped
```

接着，再制作一个名为init的Linux启动脚本，如下

```bash
#!/busybox sh
/busybox rm /init
exec /busybox sh
```





###	参考：

[1]: https://kernelnewbies.org/LinuxVersions	"LinuxNewbies"



1. [What is the difference between the following kernel Makefile terms: vmLinux, vmlinuz, vmlinux.bin, zimage & bzimage?](https://unix.stackexchange.com/questions/5518/what-is-the-difference-between-the-following-kernel-makefile-terms-vmlinux-vml)

2. [Minimal Linux](https://github.com/ivandavidov/minimal)
3. [嵌入式Linux编程 Chris Simmonds](http://www.hzcourse.com/web/refbook/detail/6937/208)
4. [从零开始构建Linux（一）](https://zhou-yuxin.github.io/articles/2015/%E4%BB%8E%E9%9B%B6%E5%BC%80%E5%A7%8B%E6%9E%84%E5%BB%BAlinux%EF%BC%88%E4%B8%80%EF%BC%89%E2%80%94%E2%80%94%E7%BC%96%E8%AF%91linux%E5%86%85%E6%A0%B8/index.html)
5. [Linux From Scratch](https://www.linuxfromscratch.org/)
6. [slax distro](https://www.slax.org/)
7. [archlinux installation guide](https://wiki.archlinux.org/title/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
8. [Why do I need initramfs?](https://unix.stackexchange.com/questions/122100/why-do-i-need-initramfs)
9. [操作系统 (2021)](http://jyywiki.cn/OS/2021/)

