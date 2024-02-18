---
layout: post
title: How to Build Linux kernel for BeagleBone Black
comments: true
---

In this tutorial, we'll walk you through the process of building a custom Linux kernel tailored specifically for the BeagleBone Black platform. By following these steps, you'll be able to create a customized embedded Linux distribution optimized for your project's requirements.


## Pre-requisite

**Host Machine Environment**

```
$ uname -a
Linux elinlinux 5.4.0-169-generic #187-Ubuntu SMP Thu Nov 23 14:53:38 UTC 2023 aarch64 aarch64 aarch64 GNU/Linux
```

**Target Machine Environment**

- Board: BeagleBone Black
- processor: Sitara XAM3359AZCZ100 Cortex A8 ARM

### FTDI USB to Serial cable

![build_kernel_result](https://hackmd.io/_uploads/H1L74Xysp.png)

Which can be found at: [http://www.ftdichip.com/Support/Documents/DataSheets/Cables/DS_TTL-232R_CABLES.pdf](http://www.ftdichip.com/Support/Documents/DataSheets/Cables/DS_TTL-232R_CABLES.pdf)

## Toolchain

### Create workspace

Create a working area to place the tools and components of the system.

```bash
cd ~
mkdir beaglebone_workspace
cd beaglebone_workspace
```

### Install require packages

Make sure that your host machine is up to date and install the essential tools, and other tools required for cross compilation.

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install build-essential
sudo apt-get install flex bison
sudo apt-get install lzop 
sudo apt-get install u-boot-tools
```

**GCC Cross Compiler**

```bash
sudo apt-get install gcc-arm-linux-gnueabihf
```

**GCC libc source**

```bash
mkdir arm-toolchain
cd arm-toolchain
wget -c https://releases.linaro.org/components/toolchain/binaries/latest-7/arm-linux-gnueabihf/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz
tar xf gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz
```

**Partition Manager**

```bash
sudo apt-get install gparted
```

# BootLoader (U-boot)

> Das U-boot source: [https://github.com/u-boot/u-boot](https://github.com/u-boot/u-boot)

### Download u-boot (u-boot-2023.10)

> Choose a u-boot version to download.

```makefile
wget -c https://github.com/u-boot/u-boot/archive/refs/tags/v2023.10.tar.gz
```

### Configure U-Boot

```makefile
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- am335x_boneblack_vboot_defconfig
```

### Compile U-boot

```makefile
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j4
```

> Refer [U-Boot: Building with GCC](https://docs.u-boot.org/en/latest/build/gcc.html) for more information.

### `MLO` and `u-boot.img`

After the `BootLoader` has been compiled, run the `ls` command to make sure that you have the executables `MLO` and `u-boot.img`. The `MLO` file is the **first stage BootLoader**, and `u-boot.img` is the **second stage BootLoader** also known as the **Bootstrap Loader**.

## Configure `Linux Kernel` for `BeagleBone Black`

> Beaglebone Linux Kernel source: [https://github.com/beagleboard/linux](https://github.com/beagleboard/linux)

### Download Linux kernel
> Choose a linux kernel version on your demand.

```bash
wget -c https://github.com/beagleboard/linux/archive/refs/tags/5.10.168-ti-r76.tar.gz
tar -xvf 5.10.168-ti-r76.tar.gz
```

### Kernel Configuration for BeagleBone Black

```bash
sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bb.org_defconfig
```

### Build the Kernel

```bash
sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- uImage dtbs LOADADDR=0x80008000 -j4
```

### Result
![build_kernel_result](https://hackmd.io/_uploads/Bk98QQkjT.png)


### Build modules

```bash
sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j4 modules
```

> Refer [admin-guide](https://github.com/beagleboard/linux/tree/v5.19/Documentation/admin-guide) for more information.
> 

# Preparing the MicroSD Card

### Create two partitions

> One partition for booting the kernel, and another one for the `Root File System`.

* Use `gparted` for partitioning.
* Create a new partition with 50 MiB for the size. Change the file system type to fat32 and label it boot.
* Add a second partition with 1000 MiB for the size, label it rootfs, and make sure that the file type is ext4.

![partition](https://hackmd.io/_uploads/HJ5CzQyoT.png)


# Root File System

### Download Busybox

```bash
cd ~/beaglebone_workspace
wget -c https://github.com/mirror/busybox/archive/refs/tags/1_36_0.tar.gz
tar -xvf 1_36_0.tar.gz
```

### Configure BusyBox

```bash
cd busybox-1_36_0
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- defconfig
```

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
```

### Enable Build static binary (no shared libs) option

![menuconfig](https://hackmd.io/_uploads/BkR0-QJop.png)

### Build BusyBox

Install Busybox into the rootfs partition of your SD card

> The build automatically generates a file "busybox.links", which is used by 'make install' to create symlinks to the BusyBox binary for all compiled in commands.  This uses the CONFIG_PREFIX environment variable to specify where to install, and installs hardlinks or symlinks depending on the configuration preferences.  (You can also manually run the install script at "applets/install.sh").
> 

> Refer [busybox: readme](https://github.com/mirror/busybox/blob/master/README) for more information.

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- CONFIG_PREFIX=/media/<username>/rootfs/ install
```

### Create required directories and files

create directories and files required by the `Linux Kernel` to boot the system

### dev/

```bash
mkdir dev
sudo mknod dev/console c 5 1
sudo mknod dev/null c 1 3
sudo mknod dev/zero c 1 5
```

### lib/ and usr/lib/

Copy the libraries from the arm toolchain into the root file system lib.

```bash
sudo cp -r ~/beaglebone_workspace/arm-toolchain/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/lib/* ./lib/
```

```bash
sudo cp -r ~/beaglebone_workspace/arm-toolchain/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/lib/* ./lib/
```

Create additional directories for mounting virtual file systems.

```bash
sudo mkdir proc sys root
```

### etc/

```bash
sudo mkdir etc etc/init.d
```

### etc/inittab

After the kernel boots, it spawns the first user process, called the `init`. `init` is a background process that runs unitil the system is shut down. This process requires a configuration file called `etc/inittab`, which contains actions the system needs to perform at a given runlevel.

```bash
sudo vi etc/inittab
```

Copy the following into the `etc/inittab` file:

```
::sysinit:/etc/init.d/rcS
::respawn:-/bin/sh
::askfirst:-/bin/sh
```
> For more setting options, please refer [IBM: inittab File](https://www.ibm.com/docs/en/aix/7.1?topic=files-inittab-file).

### etc/fstab

```bash
vi etc/fstab
```

Copy the following into `etc/fstab`:

```
proc    /proc    proc   defaults  0 0
sysfs   /sys     sysfs  defaults  0 0
```
> For more setting options, please refer [man: fstab](https://man7.org/linux/man-pages/man5/fstab.5.html).

### etc/hostname and etc/passwd

Write hostname into it.

```bash
sudo vi etc/hostname
```

```bash
sudo vi etc/passwd 
```

Copy `root::0:0:root:/root:/bin/sh`  into it.

### init.d/rcS

Create the file `init.d/rcS` to setup the system.

```bash
sudo vi etc/init.d/rcS
```

```bash
#!/bin/sh
#   ---------------------------------------------
#   Common settings
#   ---------------------------------------------
# Replace <YOUR_HOSTNAME> with your hostname!
HOSTNAME=<YOUR_HOSTNAME>
VERSION=1.0.0

hostname $HOSTNAME

#   ---------------------------------------------
#   Prints execution status.
#
#   arg1 : Execution status
#   arg2 : Continue (0) or Abort (1) on error
#   ---------------------------------------------
status ()
{
       if [ $1 -eq 0 ] ; then
               echo "[SUCCESS]"
       else
               echo "[FAILED]"

               if [ $2 -eq 1 ] ; then
                       echo "... System init aborted."
                       exit 1
               fi
       fi
}

#   ---------------------------------------------
#   Get verbose
#   ---------------------------------------------
echo ""
echo "    System initialization..."
echo ""
echo "    Hostname       : $HOSTNAME"
echo "    Filesystem     : v$VERSION"
echo ""
echo ""
echo "    Kernel release : `uname -s` `uname -r`"
echo "    Kernel version : `uname -v`"
echo ""

#   ---------------------------------------------
#   MDEV Support
#   (Requires sysfs support in the kernel)
#   ---------------------------------------------
echo -n " Mounting /proc             : "
mount -n -t proc /proc /proc
status $? 1

echo -n " Mounting /sys              : "
mount -n -t sysfs sysfs /sys
status $? 1

echo -n " Mounting /dev              : "
mount -n -t tmpfs mdev /dev
status $? 1

echo -n " Mounting /dev/pts          : "
mkdir /dev/pts
mount -t devpts devpts /dev/pts
status $? 1

echo -n " Enabling hot-plug          : "
echo "/sbin/mdev" > /proc/sys/kernel/hotplug
status $? 0

echo -n " Populating /dev            : "
mkdir /dev/input
mkdir /dev/snd

mdev -s
status $? 0

#   ---------------------------------------------
#   Mount the default file systems
#   ---------------------------------------------
echo -n " Mounting other filesystems : "
mount -a
status $? 0

#   ---------------------------------------------
#   Set PATH
#   ---------------------------------------------
export PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin

#   ---------------------------------------------
#   Start other daemons
#   ---------------------------------------------
echo -n " Starting syslogd           : "
/sbin/syslogd
status $? 0

echo -n " Starting telnetd           : "
/usr/sbin/telnetd
status $? 0

#   ---------------------------------------------
#   Done!
#   ---------------------------------------------
echo ""
echo "System initialization complete."
```

### Build directory for the kernel

After creating the above files and directories, install the kernel modules into the `Root File System`.

```bash
cd ~/beaglebone_workspace/linux-5.10.168-ti-r76
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- CONFIG_PREFIX=/media/<username>/rootfs/ modules_install
```
> Refer [admin-guide: build-directory-for-the-kernel](https://github.com/beagleboard/linux/tree/v5.19/Documentation/admin-guide#build-directory-for-the-kernel) for more information.

# Boot Partition

Create a boot folder in your workspace directory and copy `MLO`, `u-boot.img`, `uImage`, and `am335x-boneblack.dbt`.

```bash
cd ~/beaglebone_workspace/
mkdir boot
cp u-boot-2023.10/MLO boot/
cp u-boot-2023.10/u-boot.img boot/
cp linux-5.10.168-ti-r76/arch/arm/boot/uImage boot/
cp linux-5.10.168-ti-r76/arch/arm/boot/dts/am335x-boneblack.dtb boot/
```

### uEnv.txt

Go into the `boot` folder and create a file called `uEnv.txt`. This file will tell the `BootLoader` where to load the kernel image and the device binary tree.

```bash
vi boot/nEnv.txt
```

Copy the following into `uEnv.txt`:

```bash
uenv_addr=0x81000000
load_addr=0x82000000
dtb_addr=0x88000000
mmc_args=setenv bootargs console=ttyO0,115200n8 noinitrd root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait
sdboot=echo Booting from SD Card ...;mmc rescan;fatload mmc 0:1 ${uenv_addr} uEnv.txt;env import -t ${uenv_addr} $filesize;fatload mmc 0:1 ${load_addr} uImage;fatload mmc 0:1 ${dtb_addr} am335x-boneblack.dtb;run mmc_args;bootm ${load_addr} - ${dtb_addr} 
uenvcmd=run sdboot
```

**Important:** Leave a newline at the end of the `uEnv.txt` file.

Finally, copy everything from the `~/beaglebone_workspace/boot` directory to the `BOOT` partition of your SD card.

```bash
cd ~/beaglebone_workspace/boot/
sudo cp * /media/<username>/BOOT/
sync
```

# Connect FTDI cable with the board

Connect FTDI cable with J1 serial header on the board.
![J1](https://hackmd.io/_uploads/B1zStzkia.png)

> For more information, refer [BBB_SRM: 7.5 Serial Header](https://github.com/beagleboard/beaglebone-black/blob/master/BBB_SRM.pdf).

# Boot result

After you boot up the board, the system should mount the root file system successfully and you should automatically be logged in as root `#`.

![boot_result](https://hackmd.io/_uploads/B1UOuz1jT.png)


# Reference

* [How to Build a Linux Distribution for the BeagleBone Black](https://buildelinux.readthedocs.io/en/latest/embedded_systems/linux_distribution/linux_distro/)