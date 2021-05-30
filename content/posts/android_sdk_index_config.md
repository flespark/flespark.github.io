---
title: "使用vscode配置Android BSP开发环境"
date: 2021-05-30T08:05:33+08:00
draft: false
categories: ["development"]
tags: ["config", "tool", "android"]
---

身边许多同事习惯用SourceInsight阅读和修改C/C++代码，因为它资源占用低而且配置简单，导入工程就能自动生成符号索引。而且我们从头编写一套代码的情况也很少，不需要我们构建各种单元测试和配置文件，编译也是采用SDK上部署好的工具链，所以多数情况下是够用了。但这个软件已经好多年不更新了，其古早味的用户界面和圣诞卡片一般的默认语法高亮形式让人难以接受，而且还是收费软件，于是我尝试在近年雨后春笋般冒出的免费开源编辑器和IDE寻找替代品。但如果细算下来也没有多余的选择，作为BSP的我们通常要改动SDK上的各类文件，能能支持如此多种类文件和语言，又比较轻巧的编辑器就只有vim和vscode了。但是vim的配置比较复杂，学习成本太高，而且vim插件装多了打开文件时也会变慢变卡（可能是我们服务器系统的问题），远不如vscode省事，所以总结为一个新手教程。

首先，我们有两种方式把读取编译服务器上的工程，通过samba/sshfs将编译服务器的家目录映射到本地，或者使用vscode的ssh功能连接编译服务器。这里选择第二种方式，依靠vscode的remote-ssh插件和编译服务器建立连接，因为要使用/opt下的编译工具链要获取获取整个编译服务器完整的文件系统映射。插件要先下载安装，如果公司内部需要通过代理上网的，先要告诉vscode我们的代理端口。在vscode中`Ctrl+,`打开设置，点击右上角带箭头文本符号转为JSON配置文件方式打开，在最外层花括号内添加如下几行

```json
{
    "http.proxyStrictSSL": false,
    "http.proxy": "http://proxyname.domain",
    "http.proxySupport": "on",
}
```

其中`http.proxy`改为公司代理的地址，这样vscode才能联网下载插件。搜索下载`ms-vscode-remote.remote-ssh`和`ms-vscode-remote.remote-ssh-edit`两个插件，然后可以点击左侧的插件图标找到本地已安装的插件，插件安装完成后可能会提示重启vscode以刷新界面，选择Yes。顺便提一句，一开始不习惯英文界面的同学，搜索插件Chinese安装中文插件可更改界面为中文。重载后我们可以在左下角找到一个连接图标，点击选择`Remote-ssh: Connect Current Window to Host`->`Add New SSH Host`输入ssh连接命令连接到编译服务器。连接成功后vscode并不会记住我们的密码，所以自行配置使用密钥登陆ssh，ssh密钥的使用可以参考[missing semester](https://github.com/missing-semester-cn/missing-semester-cn.github.io)。所以在`Select configured SSH host or enter user@host`这一步选择`Configure SSH Hosts`->`Settings specify a custom configuration file`指定为我们的ssh客户端配置文件。连接vscode会尝试从编译服务器执行命令下载安装服务端程序vscode-server，编译服务器是没有网络接入的所以在进一步之前我们也要先告诉vscode远程连接的机器需要借用本地的互联网接入，跟前面一样在Setting.json中添加如下配置项

```json
{
    "remote.SSH.localServerDownload": "always",
    "remote.downloadExtensionsLocally": true,
    //顺便禁掉我们用不上的端口转发
    "remote.SSH.enableDynamicForwarding": false,
    "remote.SSH.enableAgentForwarding": false,
}
```

ssh连接成功后vscode会从微软的服务器下载最新版本的压缩安装文件code-stable-x64-1613044905.tar.gz安装到~/.vscode-server/bin的位置，在公司网络中，使用国际VPN的同学请注意由于网络限制可能会下载失败，切回国内VPN就能下载成功。如果使用国内VPN的同学也下载失败了，就只能选择手动安装了，这点以后补充。连接成功后，我们就可从`File`->`Add Folder to Workspace`添加目录作为工程了，vscode会根据工程中包含的文件推荐一些基础的插件，单依靠他们远无法达到SourceInsight的跳转和补全功能，必须寻找一套更为精细的配置来生成代码的符号索引。

目前使用较多的代码索引方式有ctag、cscope和compilation database，前两者和SourceInsight一样靠分析代码的语法结构生成索引，会存在跳转不够精确的情况。compilation databases（compdb）通过解析工程编译选项生成的，通常记录为一个compile_commands.json文件，能够生产精确的符号索引数据库。有许多编译系统可以直接生成这个文件（llvm，CMake，Ninja，Bazel），或者接借助工具（Build EAR，compiledb）探测make执行过程生成该文件，还有一种更灵活的方式分是编写脚本脚本从编译中间文件提取compdb。

首先这里我选择使用较多的[Build EAR工具](https://github.com/rizsotto/Bear)来生成compdb，bear通过操作系统提供的动态库预加载机制占坑wrap编译器启动编译过程中动态库调用，从而检测传入的编译参数。其次，在打开工程的时候，还需要配置[LSP](https://microsoft.github.io/language-server-protocol/specification)（Language Server Protocol）兼容的language server读取compdb生具体的符号索引，然后编辑器就能通过LSP与language server交互，提供跳转和自动补全等功能。language server这里我使用的[clangd](https://clangd.llvm.org/installation.html)，另外MaskRay大佬开发的[ccls](https://github.com/MaskRay/ccls)看起来也是一个不错的选择，他们都提供开源的跨平台能力和对多数主流编辑器的支持。所以就还要安装vscode对应的clangd插件，插件的安装非常简单，我们会发现本地和远程主机的插件是分开安装的，在连接远程主机工作的时候vscode插件中搜索clangd，选择Install会被安装到远程主机，默认安装位置是远程主机的vscode-server的extension目录中。而且`Ctrl`+`,`打开的设置界面中Remote也有独一份的配置，这里可以对于本都和远程的vscode及插件分别单独配置。

请求管理员在服务器中安装好bear和clangd之后，就可以在Makefile文件中对编译命令做手脚，在编译的同时生成compiledb。找不到管理员也没关系，clangd的安装十分简单，把官网下载的二进制放到PATH路径下就行。而bear没有提供二进制release，从源码编译的话解决依赖又很麻烦，不过可以先在相同其他相同Linux发行版系统的电脑上安装使用包管理工具安装bear，然后复制对应的二进制文件/usr/bin/bear到我们的PATH搜索目录，bear依赖于libear.so，同样要把这个文件复制出来。现在尝试编译脚本的make命令前加上bear命令，就可便触发编译的工作目录下生成compile_command.json文件。这里以Amlogic的ATV10 SDK的Linux内核代码为例，内核编译入口位于`device/amlogic/franklin/Kernel.mk`文件中，根据bear的文档这儿在make命令前添加bear命令就行，选项`-l`指定了libear.so的位置，删除编译结果输出目录然后`make bootimage`启动编译.

```diff
diff --git a/Kernel.mk b/Kernel.mk
index b6f541b..7b36cae 100644
--- a/Kernel.mk
+++ b/Kernel.mk
@@ -164,7 +164,7 @@ endef
 ##----------------------------------------------------------------------------------
 define build_kernel
        PATH=$$(cd ./$(TARGET_HOST_TOOL_PATH); pwd):$$PATH \
-               $(MAKE) -C $(KERNEL_ROOTDIR) \
+               bear -l $(HOME)/lib/libear.so $(MAKE) -C $(KERNEL_ROOTDIR) \
                O=$(realpath $(KERNEL_OUT)) \
                ARCH=$(KERNEL_ARCH) \
                CROSS_COMPILE=$(PREFIX_CROSS_COMPILE) 
```

编译完成后在SDK的根目录下生成compile_command.json文件，这个文件中可以找到每一个目标文件的编译命令。此时在vscode中File-->Open Folder打开内核代码目录，点开任何文件， 在vscode的左下角的状态栏可以找到clangd的状态指示，说明一切工作正常。

bear有一个缺点是仅探测make执行的动作，重复触发时生成的commaddb，也就是改动文件的命令对象，生成的同名文件会清除原有的内容，可以用`-a`参数指示当前为增量编译的。这里解释一下compile_command.json内容的格式，这是一个许多个命令对象组成的文件，每个命令对象指示了一个翻译单元是如何在工程中编译的。每个命令对象由主文件，执行编译的工作目录和编译命令组成，就像下面这样：

```json
[
  { "directory": "/home/user/llvm/build",
    "command": "/usr/bin/clang++ -Irelative -DSOMEDEF=\"With spaces, quotes and \\-es.\" -c -o file.o file.cc",
    "file": "file.cc" },
  ...
]
```

下面是命令对象的详细解释

- **directory**：编译的工作目录。在“command”或“file”字段中指定的所有路径都必须是该目录的相对路径或绝对路径。
- **file**：编译步骤处理的主要源文件，被language server识别作为关联源文件和编译数据库的关键字。对于同一个文件，可以有多个命令对象，例如，如果用不同的配置编译的同一个源文件。
- **command**：执行的编译命令。在JSON解转义之后，这必须是一个有效的命令，可以在构建系统基于的环境中为翻译单元重新运行正确的编译步骤。使用和shell一样的引用和解引用方式，但只有`“`和`\`作为特殊字符。不支持shell环境变量展开。
- **arguments**：将只是将command一个字符串分参数拆开了。command和arguments二选一。
- **output**：输出文件的路径，可选字段。可以用它来区分不同的编译模式（user，debug，release）。

第一次尝试编译失败了，这里给出了明确的提示，Android编译系统对mk文件中可用的外部命令作了限定

```bash
17:15:18 Disallowed PATH tool "bear" used: []string{"bear", "-l", "/home/luolinxin/lib/libear.so", "make", "-j48", "-C", "common", "O=/home/luolinxin/italy_hst/Android10.0-SDK-20201016-Hailstorm3.1/out/target/product/franklin_hybrid/ob
j/KERNEL_OBJ", "ARCH=arm64", "CROSS_COMPILE=/opt/gcc-linaro-6.3.1-2017.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-", "meson64_defconfig"}
17:15:18 See https://android.googlesource.com/platform/build/+/master/Changes.md#PATH_Tools for more information.
```

参考他给出的这个链接，在build/soong/ui/build/paths/config.go中使能bear命令解除这个报错，重新编译成功在SDK的根目录生成了compile_command.json文件

```diff
diff --git a/ui/build/paths/config.go b/ui/build/paths/config.go
index f4bb89f..2a940f0 100644
--- a/ui/build/paths/config.go
+++ b/ui/build/paths/config.go
@@ -75,6 +75,7 @@ func GetConfig(name string) PathConfig {
 
 var Configuration = map[string]PathConfig{
        "bash":     Allowed,
+       "bear":     Allowed,
        "bc":       Allowed,
        "bzip2":    Allowed,
        "date":     Allowed,
```

下面来验证一下这个文件是否有效，和clangd能否正常工作。除了vscode的clangd插件，还需要安装clangd二进制文件作为language server，很容易下载编译出二进制文件，把他放到PATH包含目录下。vscode上打开`common`目录作为工程，随便打开一个C文件，按理来说不需要额外配置，clangd会不断向上级目录查找compile_command.json文件以生成索引缓存文件，对应下边状态栏的左边会显示clangd的状态。clnagd的索引存放common的目录下的位置`.cache/index`，现在我们就可以使用它提供的语法检查，符号跳转和代码补全等功能了。

其他out-of-tree内核模块的compile_command.json的只要找到了编译入口，加上同样的命令就行。

- WIFI模块

```diff
diff --git a/wifi_bt/wifi/configs/wifi_driver.mk b/wifi_bt/wifi/configs/wifi_driver.mk
index 34acc8e..c33400a 100644
--- a/wifi_bt/wifi/configs/wifi_driver.mk
+++ b/wifi_bt/wifi/configs/wifi_driver.mk
@@ -160,7 +160,7 @@ rtl8822bs:
        cp $(shell pwd)/hardware/wifi/realtek/drivers/8822bs/rtl8822BS/8822bs.ko $(TARGET_OUT)/
 
 rtl8822cs:
-       $(MAKE) -C $(shell pwd)/$(PRODUCT_OUT)/obj/KERNEL_OBJ TopDIR=$(shell pwd)/hardware/wifi/realtek/drivers/8822cs/rtl88x2CS M=$(shell pwd)/hardware/wifi/realtek/drivers/8822cs/rtl88x2CS ARCH=$(KERNEL_ARCH) CROSS_COMPILE=$(CROSS_CO
+       bear -l $(HOME)/lib/libear.so -a $(MAKE) -C $(shell pwd)/$(PRODUCT_OUT)/obj/KERNEL_OBJ TopDIR=$(shell pwd)/hardware/wifi/realtek/drivers/8822cs/rtl88x2CS M=$(shell pwd)/hardware/wifi/realtek/drivers/8822cs/rtl88x2CS ARCH=$(KERN
        cp $(shell pwd)/hardware/wifi/realtek/drivers/8822cs/rtl88x2CS/8822cs.ko $(TARGET_OUT)/:
```

- 整个media_modules相关模块

```diff
diff --git a/Media.mk b/Media.mk
index 25e8622..01c3735 100644
--- a/Media.mk
+++ b/Media.mk
@@ -62,7 +62,7 @@ $(shell cp $(MEDIA_DRIVERS)/* $(MEDIA_MODULES) -rfa)
 
 define media-modules
        PATH=$$(cd ./$(TARGET_HOST_TOOL_PATH); pwd):$$PATH \
-               $(MAKE) -C $(KDIR) M=$(MEDIA_MODULES) ARCH=$(KERNEL_ARCH) \
+               bear  -l $(HOME)/lib/libear.so $(MAKE) -C $(KDIR) M=$(MEDIA_MODULES) ARCH=$(KERNEL_ARCH) \
                CROSS_COMPILE=$(PREFIX_CROSS_COMPILE) $(CONFIGS) \
                EXTRA_CFLAGS+=-I$(INCLUDE) modules; \
                find $(MEDIA_MODULES) -name "*.ko" | PATH=$$(cd ./$(TARGET_HOST_TOOL_PATH); pwd):$$PATH xargs -i cp {} $(MODS_OUT)
```

如果只是进行Linux开发的情况下，其实较新版本的Linux是提供了一个[gen_compile_commands.py](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=grep&q=gen_compile)脚本一次生成整个内核工程（包含out-of-tree模块）的compile_command.json，自动生成ctags和cscope的索引的工作[tags.sh](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/tags.sh)也是帮我们做好了的。
另外，基于Soong的Android编译系统也提供了对应的[选项](https://android.googlesource.com/platform/build/soong/+/HEAD/docs/compdb.md)导出所有NDK代码的关联。只需要在编译前导出环境变量：

```bash
export SOONG_GEN_COMPDB=1
export SOONG_GEN_COMPDB_DEBUG=1
```

编译同时就会在Android SDK的根目录生成compile_commands.json文件，也可`make nothing`不启动编译单独导出编译参数。使用mm编译时只会生成当前模块的compdb，同样会清空先前的compdb。对于所有命令对象都导出到SDK根目录下的compile_command.json一个文件的情况，可以在编译脚本中作调整，下面简单介绍如何使用代码检查工具帮助我们写好脚本。shellcheck可以用于检查sh脚本的潜在错误和不规范写法，配置也是同样vscode中搜索安装shellcheck插件，并在服务器的PATH包含路径下放置一个shellcheck二进制文件作为language server。shellcheck的语法检查非常严格，但对于我们编写稳健的脚本和熟悉shell的特性有益无害，不过最好还是参考[wiki](https://github.com/koalaman/shellcheck/wiki)了解如何屏蔽一些不需要的检查项和学习每个hint给出的具体原因。至于对Python和其他类型文件的支持，配置都是类似的，可用的工具同样参考missing semester的[调试及性能分析](https://missing-semester-cn.github.io/2020/debugging-profiling/)。

