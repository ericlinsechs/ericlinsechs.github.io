---
layout: post
title: "Raspberry Pi Bluetooth Speaker: A Comprehensive Guide"
comments: true
---

This guide provides a detailed walkthrough on turning your Raspberry Pi into a Bluetooth speaker. You will discover how to use Yocto for building custom Linux distributions tailored to your specific needs. In the end, you'll be able to stream music from your connected phone to your Raspberry Pi via the Bluetooth A2DP profile, with Pulseaudio managing the audio routing.

## Table of contents

- [Prerequisites](#prerequisites)
  - [Hardware](#hardware)
  - [Software](#software)
- [Setup Yocto Environment](#setup-yocto-environment)
  - [Download Required Repositories](#download-required-repositories)
- [Setup Build Environment](#setup-build-environment)
  - [Initialize Build Environment](#initialize-build-environment)
  - [Add Required Layers](#add-required-layers)
  - [Build Configuration Files](#build-configuration-files)
- [Runtime Configuration](#runtime-configuration)
  - [System](#system)
  - [PulseAudio](#pulseaudio)
  - [ALSA](#alsa)
  - [Bluetooth](#bluetooth)
  - [Optional setting](#optional-setting)
- [Debug](#debug)
  - [PulseAudio troubleshooting](#pulseaudio-troubleshooting)
- [Conclusion](#conclusion)

## Prerequisites

### Hardware
- Raspberry Pi (tested on Raspberry Pi 3B+)
- SD card
- A speaker with 3.5mm jack
- A BT USB dongle (recommended)

### Software
- Choose a Yocto release based on your demand. Please check [Yocto Releases](https://wiki.yoctoproject.org/wiki/Releases) for more details. (tested on kirkstone)
- Prepare your build host. Please see [Yocto Project Quick Build](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html#compatible-linux-distribution) and [System Requirements](https://docs.yoctoproject.org/ref-manual/system-requirements.html#system-requirements) for more info.

## Setup Yocto Environment
Before building the custom operating system image, we need to set up the Yocto environment.

```bash
cd ~/
mkdir rpi-yocto-bsp 
cd rpi-yocto-bsp/
```

### Download required repositories

```bash
mkdir sources
cd sources/

git clone -b kirkstone --depth=1 git@github.com:yoctoproject/poky.git
```
The general hardware specific BSP overlay for the RaspberryPi device.
```
git clone -b kirkstone --depth=1 https://git.yoctoproject.org/meta-raspberrypi
```
A collection of layers to suppliment OE-Core with additional packages, we will need a few layers from it.
```
git clone -b kirkstone --depth=1 https://git.openembedded.org/meta-openembedded
```

## Setup Build Environment

### Initialize build environment

```
source poky/oe-init-build-env rpi-build
```

### Add required layers
```
bitbake-layers add-layer ../sources/meta-raspberrypi
bitbake-layers add-layer ../sources/meta-openembedded/meta-python
bitbake-layers add-layer ../sources/meta-openembedded/meta-oe
bitbake-layers add-layer ../sources/meta-openembedded/meta-multimedia
```

### Build configuration files
#### conf/bblayers.conf:
```
# POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf    
# changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2" 

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \ 
  /home/eric/rpi-yocto-bsp/sources/poky/meta \
  /home/eric/rpi-yocto-bsp/sources/poky/meta-poky \
  /home/eric/rpi-yocto-bsp/sources/poky/meta-yocto-bsp \
  /home/eric/rpi-yocto-bsp/sources/meta-raspberrypi \
  /home/eric/rpi-yocto-bsp/sources/meta-openembedded/meta-oe \
  /home/eric/rpi-yocto-bsp/sources/meta-openembedded/meta-python \
  /home/eric/rpi-yocto-bsp/sources/meta-openembedded/meta-multimedia \
  "
```

#### conf/local.conf
Include the following packgages:
* **[systemd](https://wiki.archlinux.org/title/Systemd)**: a system and service manager includes [udev](https://wiki.archlinux.org/title/Udev#) reponsible for detecting ALSA audio devices on the system.
* **[PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/)**: a sound server system for transferring audio to a different machine.
* **Bluez5**: provides support for the core Bluetooth layers and protocols.
* **[packagegroup-tools-bluetooth](https://github.com/openembedded/meta-openembedded/blob/master/meta-oe/recipes-connectivity/packagegroups/packagegroup-tools-bluetooth.bb)**: includes bluetooth specific tools and PulseAudio modules for BlueZ at runtime. 

```
MACHINE ??= "raspberrypi3-64"

DISTRO_FEATURES:append = " systemd pulseaudio bluetooth alsa"
IMAGE_INSTALL:append = " pulseaudio pulseaudio-server packagegroup-tools-bluetooth alsa-utils alsa-plugins bluez5"
DISTRO_FEATURES_BACKFILL_CONSIDERED += " sysvinit"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts = "systemd-compat-units"
```

##### Note
* The `alsa-utils` package contains (among other utilities) the [alsamixer](https://man.archlinux.org/man/alsamixer.1) and [amixer](https://man.archlinux.org/man/amixer.1) utilities. `amixer` is a shell command to change audio settings, while `alsamixer` provides a more intuitive ncurses based interface for audio device configuration.
* The `alsa-plugins` package enables [upmixing/downmixing](https://wiki.archlinux.org/title/Advanced_Linux_Sound_Architecture#Upmixing/downmixing) and other advanced features for high quality resampling.
* `alsa-plugins` package is optional here.

### Deploy image

#### Build Image
```
bitbake core-image-base
```

#### Flash image to SD Card
Replace /dev/sdX with the appropriate device identifier for your SD card.
```
bzcat ./tmp/deploy/images/raspberrypi3-64/core-image-base-raspberrypi3-64.wic.bz2 | sudo dd of=/dev/sdX bs=1M conv=fsync && sync
```

## Runtime Configuration
Boot up your Raspberry Pi from the sd card.

### System
#### Check Bluetooth Controller
```
$ hciconfig
hci0:   Type: Primary  Bus: UART
        BD Address: B8:27:EB:7C:97:71  ACL MTU: 1021:8  SCO MTU: 64:1
        UP RUNNING PSCAN 
        RX bytes:1639 acl:0 sco:0 events:107 errors:0
        TX bytes:3597 acl:0 sco:0 commands:107 errors:0
```
#### Check sound card kernel module is loaded

```
# lsmod | grep snd*
snd_soc_hdmi_codec     20480  0
snd_bcm2835            28672  0
```
### PulseAudio
#### Load modules
Whenever a new sink or source appears, the `module-switch-on-connect` module will switch the default sink/source to be the new sink/source. It is optional to load this module or set default sink/source device manually.
```
$ vi /etc/pulse/default.pa
```

Add the following line:
```
load-module module-switch-on-connect
```
Refer [module-switch-on-connect](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Modules/#module-switch-on-connect) for more info.

#### Audio Playback and Latency configuration

```
vi /etc/pulse/daemon.conf
```
Add the following lines:
```
resample-method=ffmpeg
enable-remixing = no
enable-lfe-remixing = no
default-sample-format = s32le
default-sample-rate = 192000
alternate-sample-rate = 176000
default-sample-channels = 2
exit-idle-time = -1
default-fragment-size-msec = 15
```

#### Start PulseAudio daemon

```
pulseaudio --start
```

#### Check PulseAudio cards
Make sure that PulseAudio cards have been created for ALSA cards.

```
$ pactl list modules short

...
7       module-alsa-card        device_id="0" name="platform-bcm2835_audio" card_name="alsa_card.platform-bcm2835_audio" namereg_fail=false tsched=no fixed_latency_range=no ignore_dB=no deferred_volume=yes use_ucm=yes avoid_resampling=no card_properties="module-udev-detect.discovered=1"
8       module-alsa-card        device_id="1" name="platform-bcm2835_audio" card_name="alsa_card.platform-bcm2835_audio" namereg_fail=false tsched=no fixed_latency_range=no ignore_dB=no deferred_volume=yes use_ucm=yes avoid_resampling=no card_properties="module-udev-detect.discovered=1"
...
```
Refer [module-alsa-card](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Modules/#module-alsa-card) for more details.

##### Note
[module-udev-detect](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Modules/#module-udev-detect) will look for the supported cards and then select the profile you want automatically.

### ALSA
#### Unmuting the channels
By default, ALSA has all channels muted. Those have to be unmuted manually. Unmuting the sound card's master volume can be done by using amixer:
```
$ amixer sset Master unmute
```
Refer [ALSA: Unmuting the channels](https://wiki.archlinux.org/title/Advanced_Linux_Sound_Architecture#Unmuting_the_channels) for more details.

### Bluetooth
#### Start Bluetooth
```
systemctl start bluetooth
```
#### Connect Bluetooth with Phone
```
$ bluetoothctl
[bluetooth]# agent on
[bluetooth]# default-agent
[bluetooth]# discoverable on
[bluetooth]# scan on
pair XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
```
#### Check PulseAudio modules
`module-bluez5-device` is responsible for managing Bluetooth audio devices, while `module-loopback` creates a loopback audio connection, allowing audio to be routed from one source to another within the PulseAudio sound server.

Make sure these two modules are loaded:

```
$ pactl list modules short
...
22      module-bluez5-device    path=/org/bluez/hci0/dev_<MAC_ADDR> autodetect_mtu=1 output_rate_refresh_interval_ms=500 avrcp_absolute_volume=1
23      module-loopback source="bluez_source.<MAC_ADDR>.a2dp_source" source_dont_move="true" sink_input_properties="media.role=music"

```

#### List available Bluetooth A2DP audio sources
Make sure the audio source from a Bluetooth A2DP audio stream is identified. 
```
$ pactl list sources short
0       alsa_output.platform-bcm2835_audio.stereo-fallback.monitor      module-alsa-card.c      s16le 2ch 192000Hz      SUSPENDED
1       alsa_output.platform-bcm2835_audio.stereo-fallback.2.monitor    module-alsa-card.c      s16le 2ch 192000Hz      SUSPENDED
2       bluez_source.<MAC address>.a2dp_source      module-bluez5-device.c  s16le 2ch 44100Hz       SUSPENDED
```

#### List available audio sinks
```
$ pactl list sinks short
0       alsa_output.platform-bcm2835_audio.stereo-fallback      module-alsa-card.c      s16le 2ch 192000Hz      SUSPENDED
1       alsa_output.platform-bcm2835_audio.stereo-fallback.2    module-alsa-card.c      s16le 2ch 192000Hz      RUNNING
```
### Optional setting
#### Set default sink/source devices manually

```
pactl set-default-source <index|name>
pactl set-default-sink <index|name>
```

Set the card profile as audio source device:

```
pactl set-card-profile <index|name> a2dp_source
```

#### Set Volume
Adjust the volume level of an audio sink in PulseAudio to 100%. Replace <index|name> with the appropriate sink index or name.
```
pactl set-sink-volume <index|name> 100%
```

## Debug
### PulseAudio troubleshooting
For connecting error, sound glitches or any other troubles, refer [PulseAudio Bluetooth headset Troubleshooting](https://wiki.archlinux.org/title/bluetooth_headset#Troubleshooting_2).

## Conclusion
By configuring PulseAudio and connecting via Bluetooth, you can now stream music wirelessly to your Raspberry Pi speaker. 