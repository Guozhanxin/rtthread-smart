# RT-Smart Compile Instructions

[中文](README_ZH.md)|EN

Directory Intro

| Name          | Intro                                                        |
| ------------- | ------------------------------------------------------------ |
| kernel        | It is the kernel repository for rt-smart, an open source rt-thread rt-smart branch, which will be merged into RT-Thread v5.0 soon in the future |
| userapps      | User state application                                       |
| userapps/apps | It can be compiled separately, and will search for the configuration file under the parent directory (.config, rtconfig.h), generally this configuration file will be placed in the userapps directory; |
| tools         | Some Python scripts also include a front-end to kconfig      |

If it is a Windows env environment, you can perform menuconfig under userapps, configure `.config` to select AArch32 or AArch64, and toolchain GNU GCC, or musl-linux toolchain, etc.

Else, scripts for configuring environment variables in Linux/Windows environments are also provided under this repository:

- smart-env.sh - Environment variable script under Linux, configure environment variables with `source smart-env.sh` on the command line
- smart-env .bat - Environment variable script under Windows, you can run `smart-env.bat` under the command line of Env to configure environment variables

## Download Corresponding Toolchain

Please download the corresponding toolchain and expand to the `rtthread-smart/tools/gnu_gcc` directory, that can run the script of `get_toolchains.py` in the `rtthread-smart/tools` directory:

```
python3 get_toolchains.py arm
```

The following toolchain name can be arm | aarch64 | riscv64, it will automatically downloads and expands the Toolchain for Windows or Linux based on the current development Host.

## Update Submodule

Make sure that you've already updated the submodule:

```
git submodule init
git submodule update
```

Note that after the submodule is updated, it needs to switch to the `rt-smart` branch:

```
cd kernel
git checkout rt-smart

# 并And update kernel's rt-smart branch to the latest version
git pull origin rt-smart
```

## Confiure RT-Smart Development Environment

Under the repository root is a `smart-env.sh` and `smart-env.bat` script, the former for Linux and the latter for Windows.

Scripts for Linux platforms can be followed by the parameter arm | aarch64 | riscv64, actively adapting to the RT-Smart development environment. Here we're taking qemu-vexpress-a9 to verify RT-Smart on the Linux platform:

```
source ./smart-env.sh arm
```

After the execution, the relevant information will be prompted, check out if the following information is correct:

```
Arch      => arm
CC        => gcc
PREFIX    => arm-linux-musleabi-
EXEC_PATH => /mnt/d/Workspace/GitLab/rtthread-smart_wsl/tools/gnu_gcc/arm-linux-musleabi_for_x86_64-pc-linux-gnu/bin
```

## Compile User State Application

Enter `userapps` directory, run the scons:

```
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

After compiling the application, you need to put the application into the RT-Smart runtime environment (root file system), here are two methods to get that done:

1. Make a romfs, put the application into this romfs, then convert it to an array, and compile it with the kernel;
2. Put it in the SD card file system at runtime, in qemu-vexpress-a9 it is the sd.bin file;

I'm taking the first approach as an example here:

- qemu arm environment updates ROMFS source file command

```
python3 ..\tools\mkromfs.py root ..\kernel\bsp\qemu-vexpress-a9\applications\romfs.c
```

- qemu aarch64 environment updates ROMFS source file command

```
python3 ..\tools\mkromfs.py root ..\kernel\bsp\qemu-virt64-aarch64\applications\romfs.c
```

NOTE:

Currently, applications compiled under userapps are associated with the `.config`, `rtconfig.h` files in the userapps directory. `RT_NAME_MAX` in the `rtconfig.h` file is fixed at 8 bytes, and if the kernel is tuned, synchronous adjustments are also required here.

## Compile & Run

### qemu arm

#### BSP Compile

Enter the `kernel/bsp/qemu-vexpress-a9` directory and run scons and compile:

```
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

#### Run & Verify

Run `qemu.sh` or `qemu-nographic.sh` in the `/kernel/bsp/qemu-vexpress-a9` directory:

```
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

We also ran the compiled application `/bin/hello.elf` above, `hello world!` is output and then exit.

Some applications need to run in the background permanently, and you can follow it with a space and an '&' symbol, such as:

```
msh />/bin/xxx.elf &
```

### qemu aarch64

#### BSP Compile

Enter the `kernel/bsp/qemu-virt64-aarch64` directory, run scons and compile:

```
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

#### Run & Verify

Execute `qemu.bat` to enable the qemu simulation and get it run:

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