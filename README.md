# rt-thread smart版本编译说明

目录说明

![folders](folders.drawio.png)

面向开发人员，可以在Linux环境下执行prepare.py脚本，它会准备/拉取基本的仓库环境。

| 目录名 | 说明 |
| ------ | ------ |
| kernel | rt-smart的内核仓库，开源的rt-thread rt-smart分支，未来合并到RT-Thread v5.0中 |
| sdk/rt-thread | 用户态的rt-thread crt库； |
| userapps | 用户态应用程序； |
| userapps/apps | 可以进行单独编译，它会自行搜素父目录下的配置文件（.config、rtconfig.h），一般这个配置文件会放在userapps目录下； |
| userapps/gnu-apps | 外部git仓库，放置gnu的一些应用移植，例如busybox，wget, cURL等 |
| tools | 一些python脚本，也包括kconfig的前端 |

sdk/rt-thread被关联到了userapps目录下的配置（.config和rtconfig.h），所以这个库能够编译成功，需要依赖于userapps目录的配置文件。

如果是Windows的env环境可以在userapps下执行menuconfig，进行配置 `.config` 来选定AArch32还是AArch64，以及工具链GNU GCC，还是musl-linux工具链等。

另外在这个仓库下也分别提供了Linux/Windows环境下配置环境变量的脚本：

* smart-env.sh - Linux下的环境变量脚本，命令行下以`source smart-env.sh`来配置环境变量
* smart-env.bat - Windows下的环境变量脚本，可以在env的命令行下运行`smart-env.bat`来配置环境变量

## 下载对应工具链

请先下载对应的工具链并展开到`rtthread-smart/tools/gnu_gcc`目录，这部分可以在 `rtthread-smart/tools` 目录下运行 `get_toolchains.py` 的脚本：

```bash
python3 get_toolchains.py arm
```

后面的工具链名称可以是 arm | aarch64 | riscv64，它会自动根据当前的Host开发主机来下载并展开Windows或Linux的工具链。

## 更新 submodule 子模块

请确保已经更新了submodule:

```bash
git submodule init
git submodule update
```

需要注意，子模块更新下来后需要切到 `rt-smart` 分支：

```bash
cd kernel
git checkout rt-smart

# 并把kernel的rt-smart分支更新到最新版本
git pull origin rt-smart
```

## 配置 smart 开发环境

仓库根目录下有个 `smart-env.sh` 和 `smart-env.bat` 脚本，前者用于 Linux 平台，后者用于 Windows 平台。

Linux 平台的脚本后面可带参数 arm | aarch64 | riscv64，主动适配 smart 开发环境，以 Linux 平台下用 qemu-vexpress-a9 验证 smart 为例：

```bash
source ./smart-env.sh arm
```

执行完之后会提示相关信息，检查下相关信息是否正确：

```bash
Arch      => arm
CC        => gcc
PREFIX    => arm-linux-musleabi-
EXEC_PATH => /mnt/d/Workspace/GitLab/rtthread-smart_wsl/tools/gnu_gcc/arm-linux-musleabi_for_x86_64-pc-linux-gnu/bin
```

## 准备内核 SDK

内核 SDK 需要提前准备好，编译成功会生成 `sdk/rt-thread/build/static/librtthread.a` 文件。

```bash
 cd sdk/rt-thread/
 
 # 编译内核 SDK
 scons
```
还需要 install sdk：

```bash
scons install
```

除此之外，还需要编译 libdbkit ，编译成功会生成 `sdk/libdbkit/build/libdbkit.a` 文件。

```bash
 cd sdk/libdbkit/
 
 # 编译 dbkit
 scons
```
还需要 install dbkit sdk：

```bash
scons install
```

至此，内核 SDK 准备完毕。

## 编译用户态程序

进入到 `userapps` 目录中，运行 scons：

```bash
D:/Workspace/GitLab/rtthread-smart_wsl/
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

在编译完成应用程序后，需要把应用程序放到 rt-smart 运行环境（根文件系统）中，这里有两种方式：

1. 制作一份 romfs，把应用程序都放到这个 romfs 中，然后转成数组，再和内核编译在一起；
2. 把它放到运行时用的 SD 卡文件系统中，在 qemu-vexpress-a9 则是 sd.bin 文件；

我们这里为了简单起见采用第一种方式：

- qemu arm 环境更新 ROMFS 源文件命令

```bash
python3 ..\tools\mkromfs.py root ..\kernel\bsp\qemu-vexpress-a9\applications\romfs.c
```
- qemu aarch64 环境更新 ROMFS 源文件命令

```
python3 ..\tools\mkromfs.py root ..\kernel\bsp\qemu-virt64-aarch64\applications\romfs.c
```
**注意：**

目前 userapps 下编译应用会关联到 userapps 目录下的 `.config`，`rtconfig.h` 文件。在 `rtconfig.h` 文件中 `RT_NAME_MAX` 被固定在8字节大小，如果内核调整，这里也需要同步调整。

## 编译与运行

### qemu arm

#### BSP 编译

进入到 `kernel/bsp/qemu-vexpress-a9` 目录中，运行 scons 进行编译：

```bash
D:/Workspace/GitLab/rtthread-smart_wsl/kernel/bsp/qemu-vexpress-a9

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

#### 运行验证

运行 `/kernel/bsp/qemu-vexpress-a9` 目录下的 `qemu.sh` 或者 `qemu-nographic.sh` 即可：

```bash
D:/Workspace/GitLab/rtthread-smart_wsl/kernel/bsp/qemu-vexpress-a9

> ./qemu.sh

 \ | /
- RT -     Thread Smart Operating System
 / | \     5.0.0 build Oct 25 2020
 2006 - 2022 Copyright by rt-thread team
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

上面我们也运行了次编译的应用程序 `/bin/hello.elf`，看到它输出 `hello world!` 然后退出。

某些应用需要常驻后台运行的，可以在后面加一个空格和 `&` 符号，如：

```bash
msh />/bin/xxx.elf &
```

### qemu aarch64

#### BSP 编译

进入到 `kernel/bsp/qemu-virt64-aarch64` 目录中，运行 scons 进行编译：

```bash
D:\rtthread-smart\kernel\bsp\qemu-virt64-aarch64
> scons
scons: Reading SConscript files ...
scons: done reading SConscript files.
scons: Building targets ...
scons: building associated VariantDir targets: build
CC build\applications\romfs.o
LINK rtthread.elf
aarch64-linux-musleabi-objcopy -O binary rtthread.elf rtthread.bin
aarch64-linux-musleabi-size rtthread.elf
   text    data     bss     dec     hex filename
7325564    2680  102728 7430972  71633c rtthread.elf
scons: done building targets.
```

#### 运行验证

执行 `qemu.bat` 启动 qemu 模拟运行：

```
D:\rtthread-smart\kernel\bsp\qemu-virt64-aarch64  
> .\qemu.bat                                                             
qemu-system-aarch64 -M virt -cpu cortex-a53 -smp 1 -kernel rtthread.bin -nographic           
 \ | /                                                                   
- RT -     Thread Smart Operating System                                 
 / | \     5.0.0 build Jun 25 2022                                       
 2006 - 2020 Copyright by rt-thread team                                 
file system initialization done!                                         
hello rt-thread  
msh />cd bin/                                                           
msh /bin>hello.elf                                                       
msh /bin>hello world!                                                   
msh /bin>
```