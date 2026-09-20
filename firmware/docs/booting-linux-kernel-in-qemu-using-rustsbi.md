# 使用RustSBI在QEMU中直接启动Linux内核

本教程给出使用RustSBI和QEMU直接启动Linux内核的最简流程，不经过U-Boot等中间引导器。

RustSBI的**动态固件**（dynamic firmware）遵循RISC-V的`fw_dynamic`约定：上一阶段通过寄存器`a2`传递一个动态信息结构，其中包含下一阶段的入口地址和特权级。QEMU在同时指定`-bios`和`-kernel`时会按该约定填充这个结构，因此RustSBI可以直接接住QEMU装载的内核并跳转过去。这条路径比「RustSBI → U-Boot → Linux」更短，适合验证SBI固件本身是否能把真实内核带进用户态。

本教程使用软件版本如下：

|         软件          |   版本   |
| :-------------------: | :------: |
| riscv64-linux-gnu-gcc |  13.3.0  |
|  qemu-system-riscv64  |  8.2.2   |
|  RustSBI Prototyper   |  master  |
|     Linux Kernel      | 6.12.110 |
|        busybox        |  1.36.1  |

本教程的内容与CI任务`.github/scripts/prototyper-minimal-linux-boot.sh`保持一致，该脚本在每次相关改动时自动执行同样的流程。

## 安装交叉编译器和QEMU

For Ubuntu:

```shell
$ sudo apt-get update
$ sudo apt-get install -y bc bison cpio flex gcc-riscv64-linux-gnu libc6-dev-riscv64-cross libssl-dev qemu-system-misc xz-utils
```

其中`libc6-dev-riscv64-cross`提供交叉编译用的libc头文件。它只是`gcc-riscv64-linux-gnu`的推荐依赖，若使用`--no-install-recommends`安装则会被跳过，编译busybox时会因找不到`bits/libc-header-start.h`而失败。

检查交叉编译器和QEMU：

```shell
$ riscv64-linux-gnu-gcc --version
$ qemu-system-riscv64 --version
```

## 创建工作目录

```shell
$ mkdir workshop && cd workshop
```

Linux和busybox都从发布归档构建，版本在本教程中固定，与本仓库CI脚本所固定的一致。

## 编译RustSBI Prototyper

```shell
$ git clone -b main https://github.com/rustsbi/rustsbi.git
$ cd rustsbi
$ cargo prototyper build
$ cd ..
```

产物中的动态固件位于`rustsbi/target/riscv64gc-unknown-none-elf/release/rustsbi-prototyper-dynamic.elf`，后文以该文件作为QEMU的`-bios`参数。

## 编译Linux Kernel

下载并解压内核源码：

```shell
$ wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.12.110.tar.xz
$ tar -xJf linux-6.12.110.tar.xz
$ cd linux-6.12.110
```

导出环境变量并编译：

```shell
$ export ARCH=riscv
$ export CROSS_COMPILE=riscv64-linux-gnu-
$ make defconfig
$ make -j$(nproc) Image
```

RISC-V的`defconfig`已经启用了`CONFIG_BLK_DEV_INITRD`、`CONFIG_SERIAL_8250_CONSOLE`和`CONFIG_DEVTMPFS`，足以启动下面制作的initramfs，无需再手工调整配置。

编译产物为`arch/riscv/boot/Image`。编译完成后返回`workshop`目录：

```shell
$ cd ..
```

## 编译busybox

```shell
$ wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
$ tar -xjf busybox-1.36.1.tar.bz2
$ cd busybox-1.36.1
$ export ARCH=riscv
$ export CROSS_COMPILE=riscv64-linux-gnu-
$ make defconfig
```

initramfs中不含共享库，因此busybox必须静态链接。在`Settings` $\rightarrow$ `Build Options`中启用`Build static binary (no shared libs)`，或者直接修改`.config`：

```shell
$ sed -i 's/^# CONFIG_STATIC is not set$/CONFIG_STATIC=y/' .config
```

`defconfig`生成的`.config`里`CONFIG_STATIC`是被注释掉的状态，构建时直接读取`.config`，因此改写这一行即可，不需要再跑`oldconfig`。

此外还需要关闭`tc`这个applet。它依赖CBQ调度器的常量，而Linux 6.8已将该调度器从UAPI头文件中移除（busybox 1.37.0仍未适配），因此在较新的交叉工具链上会编译失败：

```shell
$ sed -i 's/^CONFIG_TC=y$/# CONFIG_TC is not set/' .config
```

initramfs的冒烟测试用不到流量控制，去掉它比固定旧版内核头文件更省事。若你使用的工具链内核头早于6.8，则无需这一步。

最后编译并安装：

```shell
$ make -j$(nproc)
$ make install
```

安装结果位于`_install/`目录，其中的`bin/busybox`是静态链接的可执行文件，其余文件是指向它的符号链接。编译完成后返回`workshop`目录：

```shell
$ cd ..
```

## 制作initramfs

initramfs会被内核解包为初始根文件系统，内核随后直接执行其中的`/init`。

```shell
$ mkdir -p rootfs
$ cp -a busybox-1.36.1/_install/. rootfs/
$ mkdir -p rootfs/proc rootfs/sys rootfs/dev
$ cat > rootfs/init << 'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
echo "RUSTSBI-SMOKE-OK $(uname -r)"
/sbin/poweroff -f
EOF
$ chmod +x rootfs/init
$ (cd rootfs && find . -print0 | cpio --null --create --format=newc --quiet | gzip -9) > initramfs.cpio.gz
```

`/init`打印的标记行用于确认用户态确实启动成功；随后`poweroff`会让QEMU退出，避免测试一直等到超时。

## 启动系统

```shell
$ qemu-system-riscv64 \
    -machine virt \
    -smp 1 \
    -m 512M \
    -nographic \
    -no-reboot \
    -bios rustsbi/target/riscv64gc-unknown-none-elf/release/rustsbi-prototyper-dynamic.elf \
    -kernel linux-6.12.110/arch/riscv/boot/Image \
    -initrd initramfs.cpio.gz
```

内核启动完成后，串口应当输出类似内容：

```
[    0.407554] Run /init as init process
RUSTSBI-SMOKE-OK 6.12.110
[    0.488517] reboot: Power down
```

看到这一行即表示：QEMU将内核交给RustSBI，RustSBI按`fw_dynamic`约定完成了M态初始化并把控制权交给S态的Linux内核，内核继而挂载initramfs并执行了用户态程序`/init`。

## 与U-Boot流程的取舍

需要验证U-Boot、发行版镜像等更完整的引导生态时，请参考[使用RustSBI & U-Boot在QEMU中启动Linux内核](./booting-linux-kernel-in-qemu-using-uboot-and-rustsbi.md)。本教程的路径不含引导器，因而启动更快、依赖更少，适合作为SBI固件本身的回归测试。
