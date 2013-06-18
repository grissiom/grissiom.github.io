---
layout: post
title: "利用 QEMU 调试 Linux 内核"
description: ""
category:
tags: [linux, qemu]
---
{% include JB/setup %}

# 编译内核

编译 ARM 内核的时候需要指定 `ARCH` 和 `CROSS_COMPILE`：

    make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi- -j4

最好在 `make menuconfig` 之前先找一个相近体系结构的 defconfig 先配置一下，否则
很难配置成功。比如可以 `make versatile_defconfig`。 ARM 体系结构的默认配置文件
在 `arch/arm/configs` 中。

# QEMU

自己编译或者从包管理器中安装即可。注意要用 `qemu-system-arm` 去执行：

    qemu-system-arm -M versatilepb \
            -kernel ~/linux-git/arch/arm/boot/zImage \
            -initrd rootfs.cpio.gz -m 64M -append "mem=64M console=ttyAMA0" \
            -gdb tcp::1234 -S -monitor stdio -serial telnet::6666,server -nographic

注意 kernel 要选择压缩过的镜像。 `-S` 代表在启动的时候等待 gdb 的连接。串口在
`telnet` 上。也是在等待外部 telnet 连接上之后才会启动程序。

qemu 启动起来之后，用 `telnet localhost 6666` 连接上去 qemu 的虚拟机就启动了，
但是这时候它还在等待 GDB 的连接，没有开始程序的运行。

# GDB 调试

上面一切都准备好之后，就可以用 GDB 连接上去调试了。在编译的过程中注意这么几个信息：

    % make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi- -j4
    ......
      LD      vmlinux
      SORTEX  vmlinux
      SYSMAP  System.map
      OBJCOPY arch/arm/boot/Image
    ......
      LD      arch/arm/boot/compressed/vmlinux
      OBJCOPY arch/arm/boot/zImage
      Kernel: arch/arm/boot/zImage is ready

这说明 `make` 会产生 `arch/arm/boot/Image` 这个未经过压缩的镜像和
`arch/arm/boot/zImage` 这个经过压缩的镜像。同时也会产生两个带内核调试信息的 ELF 文
件：`vmlinux` 和 只有压缩代码符号的 `arch/arm/boot/compressed/vmlinux`。我们在
调试的时候需要使用带内核调试信息的镜像：

```sh
    arm-none-linux-gnueabi-gdb vmlinux
```

先连接 qemu:

    (gdb) target remote :1234

然后就可以在 GDB 中调试 Linux kernel 了：

    (gdb) break init/main.c:start_kernel 
    Breakpoint 1 at 0xc037f4a4: file init/main.c, line 480.
    (gdb) c
    Continuing.

    Breakpoint 1, start_kernel () at init/main.c:480
    480             smp_setup_processor_id();

普通的 GDB 并不会很好的显示当前运行到哪一行了，用 tui 版本的 GDB 会好很多。它用
一半的屏幕显示一个代码浏览器，用高亮来标志当前要运行的行。在本例中，对应的版本
是 `arm-none-linux-gnueabi-gdbtui` 。

## 几个常用的 GDB 命令

- *list*          打印当期运行代码周围的代码。在 tui 中无效

- *n*             运行下一条语句。不会进入函数。

- *s*             运行下一条语句。会进入函数。

- *fin*           运行完当前函数

- *b*             设置断点

- *tb*            设置临时断点。当 hit 一次之后会被清除

- *p*             打印，符号的值。接受 C 语言语法的表达式。可以如 `p next` 或者 `p *next`

- *回车*          执行上一条命令。如果上一条命令为 fin 时则小心使用。
