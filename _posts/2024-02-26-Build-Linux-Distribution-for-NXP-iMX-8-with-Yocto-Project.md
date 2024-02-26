---
layout: post
title: "Build Linux Distribution for NXP i.MX 8 with Yocto Project"
comments: true
---

Welcome to our guide on building a custom Linux distribution for the NXP i.MX 8 platform using the Yocto Project. Throughout this blog, you'll learn everything from setting up your development environment to deploying the final image. Let's dive in!

## Prerequisites
Before diving into the process, it's essential to ensure that your development environment meets the necessary prerequisites. The following requirements should be in place:

### Hardware
To follow along, you'll need the i.MX 8QuadMax Multisensory Enablement Kit (MEK), which serves as our reference hardware platform. You can find more information and acquire the kit from [here](https://www.nxp.com/design/design-center/development-boards/i-mx-evaluation-and-development-boards/i-mx-8quadmax-multisensory-enablement-kit-mek:MCIMX8QM-CPU).

### Software
Ensure you have the necessary software dependencies installed:
* Choose a Yocto release based on your demand. Please check [Yocto Releases](https://wiki.yoctoproject.org/wiki/Releases) for more details.
* Prepare your build host. Please see [Yocto Project Quick Build](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html#compatible-linux-distribution) and [System Requirements](https://docs.yoctoproject.org/ref-manual/system-requirements.html#system-requirements) for more info.

### My build host environment
Here's an example of the software environment I'm using for this guide:
```
$ gcc --version
gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0

$ make --version
GNU Make 4.2.1
Built for x86_64-pc-linux-gnu

$ python -V
Python 3.8.10

$ git --version
git version 2.25.1

$ lsb_release -a
Distributor ID: Ubuntu
Description:    Ubuntu 20.04.4 LTS
Release:        20.04
Codename:       focal

$ lscpu
Architecture:                       x86_64
CPU op-mode(s):                     32-bit, 64-bit
Byte Order:                         Little Endian
Address sizes:                      46 bits physical, 48 bits virtual
CPU(s):                             224
On-line CPU(s) list:                0-223
Thread(s) per core:                 2
Core(s) per socket:                 28
Socket(s):                          4
NUMA node(s):                       4
Vendor ID:                          GenuineIntel
CPU family:                         6
Model:                              85
Model name:                         Intel(R) Xeon(R) Platinum 8180 CPU @ 2.50GHz
```

## Host Setup

### Build Host Packages
Install essential host packages on your build host using the following command:
```
sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev python3-subunit mesa-common-dev zstd liblz4-tool file locales libacl1

sudo locale-gen en_US.UTF-8
```

> For more specific package requirements, refer to the [Yocto Project Quick Build](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html#compatible-linux-distribution).

### Setting up the Repo utility

#### Repo
Repo is a tool built on top of Git, designed to manage multiple Git repositories, streamline uploads to revision control systems, and automate parts of the development workflow.

#### Yocto with Repo
Yocto leverages the Google repo tool to simplify fetching sources and setting up the build environment. It allows Yocto fetching sources from multiple location in a single command based on manifest files (xml).

#### Install
Install the “repo” utility, perform these steps:
1. Create a directory for the utility:

    ```bash
    mkdir ~/bin
    ```

2.  Download the `repo` utility:

    ```bash
    curl https://storage.googleapis.com/git-repo-downloads/repo-1 > ~/bin/repo
    chmod a+x ~/bin/repo
    ```

3. Update environment variables:

    ```bash
    export PATH=~/bin:$PATH
    ```

## Yocto Project Setup
Create a directory called `imx-yocto-bsp` for the project:
```
mkdir imx-yocto-bsp
cd imx-yocto-bsp
```

[i.MX Repo Manifest](https://github.com/nxp-imx/imx-manifest) is used to download manifests for i.MX BSP releases. Fetch the manifest for your desired release:

```
repo init -u https://github.com/nxp-imx/imx-manifest -b <branch name> [ -m <release manifest>]
```
> For more details, go check [i.MX Repo Manifest README](https://github.com/nxp-imx/imx-manifest?tab=readme-ov-file#imx-repo-manifest-readme)

In my case, I choose branch `kirkstone` and manifest `imx-5.15.71-2.2.0`:
```
repo init -u https://github.com/nxp-imx/imx-manifest -b imx-linux-kirkstone -m imx-5.15.71-2.2.0.xml
```
> If errors occur during Repo initialization, try deleting the `.repo` directory and running the Repo initialization command again.

Perform Repo synchronization, with the command periodically to update to the latest code:
```
repo sync
```

### `repo sync` error
If `repo sync` fails to clone recipes:
```
rm -rf [package path]
repo sync -f [package path]
```

## Image Build

### Setup the build folder for a BSP release
i.MX provides a script, `imx-setup-release.sh`, that simplifies the setup for i.MX machines. To use the script,
the name of the specific machine to be built for needs to be specified as well as the desired graphical backend.

The syntax for the imx-setup-release.sh script is shown below:
```
$: [MACHINE=<machine>] [DISTRO=fsl-imx-<backend>] source ./imx-setup-release.sh -b <build dir>

<machine>   defaults to `imx6qsabresd`
<backend>   Graphics backend type
    xwayland    Wayland with X11 support - default distro
    wayland     Wayland
    fb          Framebuffer (not supported for mx8)
```

Example setup for XWayland and i.MX8QM:
```
DISTRO=fsl-imx-xwayland MACHINE=imx8qmmek source imx-setup-release.sh -b ./build
```
> For more setting options, please see [i.MX Repo Manifest README](https://github.com/nxp-imx/imx-manifest?tab=readme-ov-file#setup-the-build-folder-for-a-bsp-release) and [i.MX Yocto Project User's Guide: Image Build](https://www.nxp.com/docs/en/user-guide/IMX_YOCTO_PROJECT_USERS_GUIDE.pdf) for further instructions.

### Build an image
```
bitbake <image recipe>
```
A table lists various image recipes can be found in [i.MX Yocto Project User's Guide: 5.2 Choosing an i.MX Yocto project image](https://www.nxp.com/docs/en/user-guide/IMX_YOCTO_PROJECT_USERS_GUIDE.pdf)

An example on building an image:
```
bitbake core-image-base
```

### `Bitbake` error
Here are some errors you might encounter when building image uses the bitbake command.

#### `do_fetch: Bitbake Fetcher Error: FetchError`
- Check if the remote branch exists.
- Copy the `git clone` command from log and exec it manually.
- Run `bitbake [pkg name]`
- Increase git buffer
```
git config --global http.postBuffer 1048576000
```
- For poor network connection:
The example means the remote action will block when the speed kept below 1KB/s for 600 seconds(10min):
```
git config --global http.lowSpeedLimit 1000
git config --global http.lowSpeedTime 600
```

#### `error: RPC failed; curl 18 transfer closed with outstanding read data remaining`
Switch remote URLs from HTTPS to SSH
* First, generate an SSH key. Refer [Generating a new SSH key and adding it to the ssh-agent](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key).
* Then, test your SSH connection. Refer [Testing your SSH connection](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/testing-your-ssh-connection).
* Clone repositories using SSH:
```
git clone git@github.com:OWNER/REPOSITORY.git
```
> Please check [Managing remote repositories](https://docs.github.com/en/get-started/getting-started-with-git/managing-remote-repositories#switching-remote-urls-from-https-to-ssh) for more details.

## Image Deployment
Complete filesystem images are deployed to` <build directory>/tmp/deploy/images`.  Most machine configurations provide an SD card image (.wic) and a rootfs image (.tar). The SD card image contains a partitioned image (with U-Boot, kernel, rootfs, etc.) suitable for booting the corresponding hardware.

Go to image directory:
```
cd tmp/deploy/images/imx8qmmek
```

### Flash an SD card image
An SD card image file .wic contains a partitioned image (with U-Boot, kernel, rootfs, etc.) suitable for booting the corresponding hardware. To flash an SD card image, run the following command:
```
zstdcat <image_name>.wic.zst | sudo dd of=/dev/sd<partition> bs=1M conv=fsync && sync
```
> Please see [i.MX Yocto Project User's Guide: 6 Image Deployment](https://www.nxp.com/docs/en/user-guide/IMX_YOCTO_PROJECT_USERS_GUIDE.pdf) for more information. 


## Boot Result
```
[  OK  ] Started User Login Management.
[  OK  ] Created slice Slice /system/systemd-backlight.
[  OK  ] Reached target Network.
[  OK  ] Reached target Host and Network Name Lookups.
[  OK  ] Reached target Hardware activated USB gadget.
         Starting Save/Restore Sound Card State...
         Starting Avahi mDNS/DNS-SD Stack...
         Starting Load/Save Screen … backlight:lvds_backlight@0...
         Starting Load/Save Screen … backlight:lvds_backlight@1...
         Starting Permit User Sessions...
[  OK  ] Finished Save/Restore Sound Card State.
[  OK  ] Finished Load/Save Screen …of backlight:lvds_backlight@0.
[  OK  ] Finished Load/Save Screen …of backlight:lvds_backlight@1.
[  OK  ] Finished Permit User Sessions.
[  OK  ] Started Avahi mDNS/DNS-SD Stack.
[  OK  ] Reached target Sound Card.
[  OK  ] Started Getty on tty1.
[  OK  ] Started Serial Getty on ttyLP0.
[  OK  ] Reached target Login Prompts.
[  OK  ] Reached target Multi-User System.
         Starting Record Runlevel Change in UTMP...
[  OK  ] Finished Record Runlevel Change in UTMP.


NXP i.MX Release Distro 5.15-kirkstone imx8qmmek ttyLP0

imx8qmmek login: root
root@imx8qmmek:~#
```

## Reference
* [i.MX Yocto Project User's Guide](https://www.nxp.com/docs/en/user-guide/IMX_YOCTO_PROJECT_USERS_GUIDE.pdf)
* [Yocto Project Documentation](https://docs.yoctoproject.org/4.3.2/index.html)
* [bitbake user manual intro](https://docs.yoctoproject.org/bitbake/bitbake-user-manual/bitbake-user-manual-intro.html#concepts)
* [error: RPC failed; curl transfer closed with outstanding read data remaining](https://stackoverflow.com/questions/38618885/error-rpc-failed-curl-transfer-closed-with-outstanding-read-data-remaining)