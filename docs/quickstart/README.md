# RT-Thread Smart快速上手

RT-Thread Smart（简称rt-smart）是基于RT-Thread操作系统衍生的新分支，面向带MMU，中高端应用的芯片，例如ARM Cortex-A系列芯片，MIPS芯片，带MMU的RISC-V芯片等。

本软件包是RT-Thread Smart的示范软件包，可运行在QEMU模拟出的VExpress-A9机器中。

## 目录说明

| 目录名 | 说明 |
| ------ | ------ |
| kernel | RT-Thread内核 (打开RT_USING_SMART将开启smart特性) |
| tools | 编译rt-smart时用到的python脚本 |
| userapps | 用户态开发环境和示例 |
| userapps/apps | 用户态应用示例 |
| userapps/linker_scripts | 不同CPU体系结构下的默认链接脚本 |
| userapps/sdk | 用户态环境中需要用到的头文件，库等 |

如果是Windows的env环境可以在userapps下执行menuconfig，进行配置.config，来选定GNU GCC，还是musl-linux工具链。

## 编译验证

在Windows下请下载[env工具](https://www.rt-thread.org/page/download.html)。下载env工具，可以正常使用后，请打开终端窗口，然后切换到这个代码包根目录，**运行smart-env.bat**，它会设置一定的环境变量，然后整体的smart开发环境就可以使用了。

```bash
> cd \workspace\rt-smart
> smart-env.bat
```

**注**

此处运行smart-env.bat以设置环境，这步非常重要，它包括编译器设置。同时它也会设置工具链的前缀，可以在env终端下输入`set RTT_CC_PREFIX`命令看看返回结果是否生效：

```bash
> set RTT_CC_PREFIX
RTT_CC_PREFIX=arm-linux-musleabi-
```

### 编译内核

进入到`rt-smart/kernel/bsp/qemu-vexpress-a9`目录中，运行scons

```bash
D:\workspace\rt-smart\kernel\bsp\qemu-vexpress-a9
> scons
scons: Reading SConscript files ...
scons: done reading SConscript files.
scons: Building targets ...
scons: building associated VariantDir targets: build
CC build\kernel\src\thread.o
LINK rtthread.elf
arm-linux-musleabi-objcopy -O binary rtthread.elf rtthread.bin
arm-linux-musleabi-size rtthread.elf
   text    data     bss     dec     hex filename
 894824   40644  122416 1057884  10245c rtthread.elf
scons: done building targets.
```

### 编译应用程序

进入到`rt-smart/userapps`目录中，运行scons

```bash
D:\workspace\rt-smart
> cd userapps
> scons
scons: Reading SConscript files ...
scons: done reading SConscript files.
scons: Building targets ...
CC build\hello\main.o
CC build\ping\main.o
CC build\pong\main.o
CC build\vi\optparse-v1.0.0\optparse.o
CC build\vi\vi.o
CC build\vi\vi_utils.o
LINK root\bin\hello.elf
LINK root\bin\ping.elf
LINK root\bin\pong.elf
LINK root\bin\vi.elf
scons: done building targets.
```

在编译完成应用程序后，需要把应用程序放到rt-smart运行环境（根文件系统）中，这里有两种方式：

1. 制作一份romfs，把应用程序都放到这个romfs中，然后转成数组，再和内核编译在一起；
2. 把它放到运行时用的SD卡文件系统中，在qemu-vexpress-a9则是sd.bin文件；

我们这里为了简单起见采用第一种方式：

```bash
python ..\tools\mkromfs.py root ..\kernel\bsp\qemu-vexpress-a9\applications\romfs.c
```

然后再重新编译下内核让romfs生效：

``` bash
D:\workspace\rt-smart\kernel\bsp\qemu-vexpress-a9
> scons
scons: Reading SConscript files ...
scons: done reading SConscript files.
scons: Building targets ...
scons: building associated VariantDir targets: build
CC build\applications\romfs.o
LINK rtthread.elf
arm-linux-musleabi-objcopy -O binary rtthread.elf rtthread.bin
arm-linux-musleabi-size rtthread.elf
   text    data     bss     dec     hex filename
1903828   40644  122416 2066888  1f89c8 rtthread.elf
scons: done building targets.
```

**注**

  目前userapps下编译应用会关联到userapps目录下的`.config`，`rtconfig.h`文件。在`rtconfig.h`文件中`RT_NAME_MAX`被固定在8字节大小，如果内核调整，这里也需要同步调整。

## 运行

在env工具环境下，直接使用qemu运行即可：

```bash
D:\workspace\rt-smart\kernel\bsp\qemu-vexpress-a9
> qemu

 \ | /
- RT -     Thread Smart Operating System
 / | \     5.0.0 build Oct 25 2020
 2006 - 2020 Copyright by rt-thread team
lwIP-2.0.2 initialized!
try to allocate fb... | w - 640, h - 480 | done!
fb => 0x61100000
[I/sal.skt] Socket Abstraction Layer initialize success.
file system initialization done!
hello rt-thread
[I/SDIO] SD card capacity 65536 KB.
[I/SDIO] switching card to high speed failed!
msh />
msh />/bin/hello.elf
hello world!
```

上面我们也运行了次编译的应用程序`/bin/hello.elf`，看到它输出`hello world!`然后退出。
