## Introduction

Debian and its derivatives are a large family of Linux operating systems. This includes Debian, Ubuntu, Xubuntu, Kubuntu, and many specialized distributions. All distributions share the same package management structure. Packages available for one version are very likely to be compatible with other versions.

The Debian family of operating systems are very versatile. As a result there are many ways to install and set-up one of these systems.

Debian focuses on stability and security of its releases. As a result, new features are not added to existing installations, and new releases are not frequent. Software packages are updated for security releases only. There are also testing and unstable versions. These are development versions, which could become the next main version. Testing and unstable versions are updated frequently, but may be unstable.

## Official SolidRun Images

### Support Matrix

| Release | Support | Window-Systems for Multimedia | Accelerated Desktops |
| --- | --- | --- | --- |
| Buster | Yes | Framebuffer, X11 | Mate |
| Stretch | Yes | Framebuffer, Wayland | Weston |
| Jessie | Yes | Framebuffer, X11 | Mate |
| Wheezy | Expired | Framebuffer, X11 | XFCE |

### Download and Install

All images are available for download at https://images.solid-run.com/IMX6/Debian. Please scroll to the bottom to find a log of important changes.
Several tools are available for writing them to block storage, including etcher.io, win32diskimager and dd. Please make sure to decompress them first!

Installation also works through the Ignition installer: [Flash Ignition to an SD card](https://solidrun.atlassian.net/wiki/spaces/developer/pages/288129025/Flashing+an+SD+Card), then boot it on the device to download and install the latest Debian automatically.
**Note:** Solid-Run stopped supporting "ignition", you can try the Debian/Yocto articles in the developer site.

### Install to eMMC

If you have the optional eMMC on your SOM use these instructions to install Debian on the eMMC and boot from there.

1. Set the [boot select jumpers](https://solidrun.atlassian.net/wiki/spaces/developer/pages/286621835/HummingBoard+Edge+Gate+Boot+Jumpers) to SD card

2. Boot from SD

3. Download the Debian image:

       wget https://images.solid-run.com/IMX6/Debian/sr-imx6-debian-buster-cli-20190906.img.xz

4. As root write the image to the eMMC:

       xz -dc sr-imx6-debian-buster-cli-20190906.img.xz | dd of=/dev/mmcblk2 bs=4M conv=fsync

5. Download bootloader images:

       wget https://images.solid-run.com/IMX6/U-Boot/spl-imx6-sdhc.bin
       wget https://images.solid-run.com/IMX6/U-Boot/u-boot-imx6-sdhc.img

6. As root write the bootloader images to the eMMC:

       dd if=spl-imx6-sdhc.bin of=/dev/mmcblk2 bs=1K seek=1 conv=fdatasync
       dd if=u-boot-imx6-sdhc.img of=/dev/mmcblk2 bs=1K seek=69 conv=fdatasync

   Ensure eMMC Data Partition is selected as boot source:

       # EITHER On Linux (command from mmc-utils package)
       mmc bootpart enable 7 0 /dev/mmcblk2
       # OR On U-Boot
       mmc partconf 1 1 7 0

7. Shut the system down with the `poweroff` command

8. Disconnect power source

9. Set the boot select jumpers to eMMC boot

10. Boot the system from eMMC

### Get Started

Once you are greeted by a login prompt, the default username and password are both "debian". For security reasons there is **no** root password! If you really need one, you can run `sudo passwd root` to set your own.

### Networking

Connman comes preinstalled on all images to facilitate easier network management, especially for wifi and tethering. Please refer to the [documentation](https://01.org/connman/documentation) for more information.

### Accelerated OpenGL-ES

The drivers for the Vivante GPU that is part of i.MX6 SoCs are available as packages from our repository to be installed with apt. There are variants for Framebuffer, Wayland and X11:

- Framebuffer:
  - runtime: `imx-gpu-viv imx-gpu-viv-fb`
  - development: `imx-gpu-viv-dev imx-gpu-viv-fb-dev`
- Wayland (TBD)
- X11
  - driver: `xserver-xorg-video-imx-viv`
  - runtime: `imx-gpu-viv imx-gpu-viv-x11`
  - development: `imx-gpu-viv-dev imx-gpu-viv-x11-dev`

When all variants are installed side by side, the default is selected through `update-alternatives` by configuring the `vivante-gal` link group:

    sudo update-alternatives --config vivante-gal

#### Demos
- eglinfo

  A small application for printing version and feature information of the active EGL and OpenGl-ES implementation. It is available [here on github](https://github.com/dv1/eglinfo) and installable through apt from our bsp repository:

      apt install eglinfo-fb
      # or any of eglinfo-wl eglinfo-x11

- glmark2

  Benchmark for OpenGL-ES 2.0 - available [here on github](https://github.com/glmark2/glmark2).
  Instructions for building and running from source:

      sudo apt install build-essential git imx-gpu-viv-fb-dev imx-gpu-viv-dev libjpeg-dev libpng-dev pkg-config python
      git clone -b fbdev https://github.com/Josua-SR/glmark2.git
      cd glmark2
      ./waf configure --with-flavors=fbdev-glesv2
      ./waf build -j4
      sudo ./waf install

      glmark2-es2-fbdev

### Accelerated Video Decoding and Playback

While directly using the VPU and IPU libraries provided by NXP is possible, that use-case goes beyond the scope of this document.
Instead we are using the [GStreamer 1.0 plugins for i.MX platforms](https://github.com/Freescale/gstreamer-imx) to make use of those acceleration blocks in the SoC.

There are variants for Framebuffer, Wayland and X11:
- `gstreamer1.0-imx-fb`
- `gstreamer1.0-imx-wl`
- `gstreamer1.0-imx-x11`

Alike the GPU userspace, when more than one variant is installed at a time, the desired graphics system can be chosen through `update-alternatives`:

    update-alternatives --config gst1.0-imx

#### Examples

The examples below depend on a number of additional gstreamer elements and utilities:

    sudo apt install gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-alsa gstreamer1.0-tools
    sudo apt install gstreamer1.0-imx-fb

And the [Project Peach](https://peach.blender.org/) [big buck bunny 1080p 30Hz](http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4) as demo.

- automatic with playbin element

      gst-launch-1.0 playbin uri=http://distribution.bbb3d.renderfarming.net/video/mp4/bbb_sunflower_1080p_30fps_normal.mp4

- automatic with playbin (720p)

      gst-launch-1.0 playbin uri=http://distribution.bbb3d.renderfarming.net/video/mp4/big_buck_bunny_720p_surround.avi

- explicit vpu + g2d - video only

      gst-launch-1.0 \
        filesrc location=bbb_sunflower_1080p_30fps_normal.mp4 ! \
        qtdemux ! h264parse ! imxvpudec ! \
        imxg2dvideosink

- explicit a52dec + alsa + sgtl5k (3.5mm audio jack) - audio only

      gst-launch-1.0 \
        filesrc location=bbb_sunflower_1080p_30fps_normal.mp4 ! \
        qtdemux ! ac3parse ! a52dec ! \
        alsasink device=default:CARD=Codec

## Customize

### Using custom Device-Tree Blobs

Until recently the system would replace any custom DTB files in /boot on upgrade of the kernel package. Now there is a way to get rid of this behaviour:

#### Images after 25/08/2016

The DTBs managed by the Distribution are now located under /boot/dtb-<VERSION>. Custom ones can be placed directly in /boot. That is now the place where u-boot looks first!

#### Images before 25/08/2016

These images did not have U-Boot patched appropriately. To use anyway, a few dangerous steps need to be performed:

1. Install latest U-Boot package:

       apt-get update && apt-get upgrade

   **Make sure that the u-boot package has at least version 2013.10.1464952045-1! The version can be checked by running `dpkg -l u-boot-cubox-i`**

2. Copy U-Boot to the bootsector:
   Follow the instructions available on [U-Boot](https://solidrun.atlassian.net/wiki/spaces/developer/pages/287179374/i.MX6+U-Boot), section **Writing U-Boot to the SD card**. Be sure to do this on the device itself, using `cubox-i-spl` and `u-boot.img` in `/boot`, to `/dev/mmcblk0`.

3. Wipe U-Boot Environment:
   reboot, and press a key to activate u-boot console. Then run:

       env default -a
       saveenv
       reset

4. Finally delete the transitional package `dtb-subfolder-compat`:

       sudo apt-get remove dtb-subfolder-compat

   **This will remove kernel-3.14.y-fslc-imx6-sr. Make sure that the replacement linux-image-3.14.y-fslc-imx6-sr was already installed.**

### Fork board-support packages

In order to properly support the i.MX6 including all of its multimedia capabilities, many custom debs that are not part of Debian have been created. These include an optimized kernel, the Vivante GPU userspace, gstreamer plugins and many more. There are 2 ways for getting the package sources:

1. use apt-get source, e.g.: `apt-get source linux-image-4.9.y-imx6-sr`

2. browse the [SolidRun github](https://github.com/SolidRun), and the old [packaging organization](https://github.com/mxOBS)

All of our packages can be built by invoking dpkg-buildpackage directly, or through git-buildpackage. If there are specific questions regarding packaging of customizations please contact us.

Also note that [our instance](https://obs.solid-build.xyz/) of the [OpenBuildService](https://openbuildservice.org/) which we use to automate builds is available to the public on request.

## Pure Debian (upstream)

Our long-term goal is enabling as much of our hardware upstream as part of Debian where feasible. Success is highly dependent on the efforts of the general community around the i.MX6 SoCs, as well as the amount of proprietary blobs involved. At this point upstream support for the i.MX6 is pretty good, and there is ongoing work on a free graphics stack for the Vivante GPU. However the performance of video de- and encoding as well as OpenGL rendering is nowhere near to the proprietary solutions currently used.

The instructions below are a subset of [chapter 4](https://www.debian.org/releases/stable/armhf/ch04.en.html) from the [Debian GNU/Linux Installation Guide](https://www.debian.org/releases/stable/armhf/).

1. Create bootable installer
   - network install

         wget http://deb.debian.org/debian/dists/buster/main/installer-armhf/current/images/netboot/SD-card-images/firmware.MX6_Cubox-i.img.gz
         wget http://deb.debian.org/debian/dists/buster/main/installer-armhf/current/images/netboot/SD-card-images/partition.img.gz
         zcat firmware.MX6_Cubox-i.img.gz partition.img.gz > installer.img

   - offline install

         wget http://deb.debian.org/debian/dists/buster/main/installer-armhf/current/images/hd-media/SD-card-images/firmware.MX6_Cubox-i.img.gz
         wget http://deb.debian.org/debian/dists/buster/main/installer-armhf/current/images/hd-media/SD-card-images/partition.img.gz
         zcat firmware.MX6_Cubox-i.img.gz partition.img.gz > installer.img

2. Write installer image block device

   We recommend using [etcher.io](https://www.balena.io/etcher/) for writing the `installer.img` file created in the previous step to a microSD or USB drive.

   Note: When choosing USB, a version of U-Boot must be already installed on the device.

3. Write Debian Installation ISO to a **different** USB drive (for offline installation only)

   Download [debian-10.5.0-armhf-xfce-CD-1.iso](https://cdimage.debian.org/debian-cd/current/armhf/iso-cd/debian-10.5.0-armhf-xfce-CD-1.iso) and place it as a file on a USB drive formatted with a filesystem supported by Debian, such as ext4.

4. Perform Installation

   - Attach the bootable installer media from step 2 to the device
   - Optional: Attach the drive created in step 3 to the device
   - connect to the serial console
   - power on and walk through the installation prompts

   Note: It is safe to overwrite the bootable installer media from step 2 during the installation.

5. Post-Installation Tweaks

   - (Re-) Install U-Boot if necessary

     If the boot media has been used as install target, U-Boot has to be reinstalled as documented on our [U-Boot page](https://solidrun.atlassian.net/wiki/spaces/developer/pages/287179374/i.MX6+U-Boot). Both the [SolidRun U-Boot builds](https://images.solid-run.com/IMX6/U-Boot/) and [those from Debian](http://debian.backend.mirrors.debian.org/debian/dists/buster/main/installer-armhf/current/images/u-boot/MX6_Cubox-i/) are usable.

   - Enable Non-Free packages

     A number of components require proprietary firmware to operate. Please refer to the [Debian Wiki](https://wiki.debian.org/SourcesList) for enabling the `non-free` component.

   - Broadcom WiFi:
     1. Install firmware package: `apt install firmware-brcm80211`

        Note: The firmware for BCM4330 is actually outdated and has poor performance. We advise directly grabbing `brcmfmac4330-sdio.bin` from [our github](https://github.com/SolidRun/deb-pkg_cuboxi-firmware-wireless) and installing it to `/lib/firmware/brcm/` instead.

     2. Install chip configuration:

            wget https://github.com/SolidRun/deb-pkg_cuboxi-firmware-wireless/raw/master/brcmfmac4329-sdio.txt
            wget https://github.com/SolidRun/deb-pkg_cuboxi-firmware-wireless/raw/master/brcmfmac4330-sdio.txt
            install -v -m755 -o root -g root brcmfmac4329-sdio.txt brcmfmac4330-sdio.txt /lib/firmware/brcm/

   - Ti WiFi (i.MX6 SoM v1.5 and later):

     1. Install firmware package: `apt install firmware-ti-connectivity`

     2. Install chip configuration

            wget https://github.com/SolidRun/deb-pkg_cuboxi-firmware-wireless/raw/master/wl18xx-conf.bin
            install -v -m755 -o root -g root wl18xx-conf.bin /lib/firmware/ti-connectivity/

   - Analogue Audio (sgtl5k)

     1. Install firmware package:  `apt install firmware-misc-nonfree`

     2. Install an initramfs-tools hook for including sdma firmware in initramfs:

            wget https://github.com/SolidRun/pkg-bsp/raw/master/initramfs-tools/imx-sdma.sh
            install -m755 -o root -g root imx-sdma.sh /etc/initramfs-tools/hooks/imx-sdma
            update-initramfs -u

## Known Issues

### wl1271_sdio mmc0:0001:2: wl12xx_sdio_power_on: failed to get_sync(-13)

The Ti WiFi on SoMs v 1.5 does not currently work with the 4.19 kernel in Debian.
Newer releases of Linux appear to have solved the problem - e.g. 5.7.10 is known to work - and can be installed from [buster-backports](https://backports.debian.org/Instructions/) by `apt install -t buster-backports linux-image-armmp`.

### Debian Installer does not drive the display

As of Debian Buster, the debian-installer does not include the kernel modules necessary for driving the HDMI port. Therefore the serial console has to be used to perform the installation.

### apt upgrade – The following packages have been kept back

#### bsp-cuboxi (bsp-solidrun-imx6)

The package bsp-cuboxi pulling in basic configuration and firmware files has been renamed to bsp-solidrun-imx6. At the same time some firmware files have have landed in upstream debian, stretch-backports in partciular; a situation that apt does not solve without human intervention.

You know that you have run into this situation if apt upgrade output looks similar to this:

    debian@sr-imx6:~$ sudo apt upgrade
    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    Calculating upgrade... Done
    The following packages were automatically installed and are no longer required:
      bsp-cuboxi cuboxi-firmware-wireless cuboxi-firmware-wireless-bluetooth cuboxi-firmware-wireless-bluetooth-ti
      cuboxi-firmware-wireless-wifi cuboxi-firmware-wireless-wifi-config cuboxi-firmware-wireless-wifi-config-ti
      firmware-ti-connectivity libg2d1.1
    Use 'sudo apt autoremove' to remove them.
    The following packages have been kept back:
      bsp-cuboxi
    0 upgraded, 0 newly installed, 0 to remove and 1 not upgraded.

The solution is rather simple: force the upgrade for bsp-cuboxi to its transitional version – and tell apt to resolve new dependencies from backports:

    debian@sr-imx6:~$ sudo apt install -t stretch-backports bsp-cuboxi
    Reading package lists... Done
    Building dependency tree       
    Reading state information... Done
    The following packages were automatically installed and are no longer required:
      cuboxi-firmware-wireless cuboxi-firmware-wireless-bluetooth libg2d1.1
    Use 'sudo apt autoremove' to remove them.
    The following additional packages will be installed:
      bsp-solidrun-imx6 firmware-misc-nonfree
    The following NEW packages will be installed:
      bsp-solidrun-imx6 firmware-misc-nonfree
    The following packages will be upgraded:
      bsp-cuboxi
    1 upgraded, 2 newly installed, 0 to remove and 48 not upgraded.
    1 not fully installed or removed.
