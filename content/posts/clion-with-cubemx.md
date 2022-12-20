---
title: "Clion With Cubemx"
date: 2022-12-17T21:41:44+08:00
draft: true
---

###	配置Clion用于STM32开发

参考稚辉君的分享：[配置CLion用于STM32开发【优雅の嵌入式开发】 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/145801160)，同时有如下改进和注意事项：

1. 使用scoop安装MinGW，openocd，gcc-arm-none-eabi-gcc更加方便；

2. 从Clion打开CubeMX生成代码时，首先会生成一套默认芯片的项目配置文件。这里存在一个bug，导致会面CudeMX不会生成所选择芯片的完整项目配置。所以选择芯片型号前要先把Clion工程目录下的除ioc的默认配置都删除，然后在CudeMX中完成配置，点击"GENERATE CODE"才会重新生成正确的配置文件。注意CubeMX项目保存目录和Clion要设置为同一目录，不用在"Project Manager"中更改直接选择"File"->"Save Project"点"OK"即可。

3. CubeMX配置生成完毕返回Clion会提示选择openocd的cfg配置文件，这里一定要选一个文件，然后再更改，不然编译目标中不会出现openocd。

4. 可以从STM32官网下载芯片的svd(System view description)文件，在Clion中配置好方便调试。

5. 在代码中使用任何的libc接口，例如printf将会在链接时引入专为bare-metal嵌入式系统设计的[newlib](https://sourceware.org/newlib/libc.html)库，默认使用的[gcc specs文件](https://gcc.gnu.org/onlinedocs/gcc/Spec-Files.html)使用的静态库文件对于中低端MCU而言依然体积过大，所以在CMakeLists.txt中修改编译使用的specs文件：
   ```cmake
   add_link_options(--specs=nano.specs)
   ```

   可以大幅减少生成固件的体积。

   
