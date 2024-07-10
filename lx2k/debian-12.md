# Debian 12 on SolidRun LX216x platforms

Disclaimer: Support is provided by the Debian community. We encourage engaging with them.

## Status

| Platform              | Status      |
| --------------------- | ----------- |
| Honeycomb Workstation | Supported   |
| Clearfog LX2-Lite     | In Progress |

### Open Issues

- [linux-image-6.1.0-17-arm64: please enable support for lx2160a serdes runtime configuration](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1061117):

  Breaks all native network interfaces on LX216x SoCs.

  Closed since 6.8.9-1.

  - http://deb.debian.org/debian/pool/main/l/linux-signed-arm64/linux-image-arm64_6.9.8-1_arm64.deb
  - http://deb.debian.org/debian/pool/main/l/linux-signed-arm64/linux-image-6.9.8-arm64_6.9.8-1_arm64.deb

- clearfog lx2-lite rj45 ports wrongly report 100Mbps link

  The board uses a marvell 88e2580 octa-port phy which currently has no driver in the kernel.

## Install U-Boot (One-Time Setup)

To get started, minimal bootable disk images for microSD can be downloaded from [images.solid-run.com](https://images.solid-run.com/LX2k/lx2160a_build).
The latest disk image named `lx2160acex7_2000_700_????_8_5_2-*******.img.xz` with the first number lower or equal to the installed ram speed should generally work.

Decompress the `lx2160acex7_*.img.xz` file, then write the bootloader part to an SD card, e.g.:

    sudo dd if=/dev/zero bs=512 count=1 of=/dev/sdX
    sudo dd bs=512 seek=1 skip=1 count=131071 conv=fsync if=lx2160acex7_2000_700_2400_8_5_2-bae3e6e.img of=/dev/sdX

Configure the DIP switches for SD boot: `0 1 1 1 X` (0 = off, 1 = on, x = don't care)

As long as it is not overriden (e.g. by installing Debian to the Card), it can be used as bootloader.

Alternatively while first booting from this card, bootloader can be installed permanently to SPI flash on the CEX module:

### Install U-Boot to SPI Flash

1. Interrupt boot process at the timeout promp by pressing any key:

       fsl-mc: Booting Management Complex ... SUCCESS
       fsl-mc: Management Complex booted (version: 10.37.0, boot status: 0x1)
       Hit any key to stop autoboot:  0
       =>

2. Load boot-loader parts from sd-card

       mmc dev 0
       mmc read 0x81100000 0 0x20000

3. write boot-loader to spi flash

       sf probe
       sf erase 0 0x4000000
       sf update 0x81101000 0 0x20000
       sf update 0x81120000 0x20000 0x3FE0000

4. change DIP switches for SPI boot: `0 0 0 0 X` (0 = off, 1 = on, x = don't care)

5. remove sd-card, power-cycle device and confirm that u-boot is starting again (see same prompt as in step 1).

## Prepare Debian Install Media

Debian provides a special net-install medium compatible with U-Boot.
It can be found on the [Debian Website](https://www.debian.org/) by following "Other downloads", "Download an installation image", "A small installation image", "
Tiny CDs, flexible USB sticks, etc.", "arm64", "SD-card-images": https://deb.debian.org/debian/dists/bookworm/main/installer-arm64/current/images/netboot/SD-card-images/

The commands below can create a generic bootable disk at fictonal `/dev/sdX` block device (SD-Card or USB Flash-Drive).
Make sure to replace X with the name or number of your destination drive. *Carelessness in this step can lead to data loss!*

    wget https://deb.debian.org/debian/dists/bookworm/main/installer-arm64/current/images/netboot/SD-card-images/firmware.none.img.gz
    wget https://deb.debian.org/debian/dists/bookworm/main/installer-arm64/current/images/netboot/SD-card-images/partition.img.gz
    zcat firmware.none.img.gz partition.img.gz | sudo dd of=/dev/sdX bs=4M conv=fsync

## Boot Debian Installer

### Preparation (One-Time Setup)

Connect the serial console, power-on the device and interrupt automatic boot as the timeout prompt appears:

    ...
    fsl-mc: Booting Management Complex ... SUCCESS
    fsl-mc: Management Complex booted (version: 10.28.1, boot status: 0x1)
    Hit any key to stop autoboot:  0
    =>

Optionally clear all current settings:

    => env default -a

Ensure `fdtfile` variable includes `freescale/` prefix:

    => print fdtfile
    fdtfile=fsl-lx2160a-clearfog-cx.dtb
    => setenv fdtfile freescale/fsl-lx2160a-clearfog-cx.dtb

Enable the smmu bypass:

    => setenv bootargs arm-smmu.disable-bypass=0

Save these settings for future (re-)boots:

    => saveenv

IF setting were cleared in earlier step, use reset-button or power-cycle the unit now.

### Boot Debian Installer

As long as the bootloader is not overriden / reinstalled, the steps above
are required only a single time.
The steps below are required every time the debian installer is invoked,
first install and reinstall / repair.

Stop the watchdog:

    => wdt dev watchdog@23a0000
    => wdt stop

Boot the install media:

    setenv boot_targets usb0
    # or alternatively microsd
    # setenv boot_targets mmc0
    boot

Continue going through the installation menus - and remove install media when it has completed.

## First Reboot

In case there might be multiple media with bootable operating systems present at the same time,
e.g. on both SD-Card and eMMC, the default boot-order should be reviewed and customised.
This is controlled through the boot_targets u-boot variable, and can be customised - e.g. to prefer eMMC over microSD:

    => print boot_targets
    boot_targets=usb0 mmc0 mmc1 scsi0 nvme0 dhcp
    => setenv boot_targets usb0 mmc1 mmc0 nvme0 scsi0
    => saveenv
    Saving Environment to MMC... Writing to MMC(0)... OK

Finally boot the new system by either pressing the reset button, or using the `boot` command.

## Tweaks

### Enable Fan-Controller

**Note: Bugfix has been applied and is included in Debian since 6.1.0-20-arm64 (Linux 6.1.84+)**

If the system overheats at the default low fan-speed under load,
likely the fan-controller driver wasn't loaded automatically.
After loading the driver, fan-speed should dynamically follow cpu temperature:

    sudo modprobe amc6821

A bugfix has been submitted to the kernel: [hwmon: (amc6821) add of_match table](https://patchwork.kernel.org/project/linux-hwmon/patch/20240307-amc6821-of-match-v1-1-5f40464a3110@solid-run.com/)

### Enable SFP Ports

The SFP ports network devices (`eth*`) can be started manually after each (re-) boot using NXP's DPAA2 Management Utility [restool](https://github.com/nxp-qoriq/restool).

#### Patch Debian Kernel

Unfortunately Debian is missing an essential driver for the SerDes link between SoC and SFPs:

    CONFIG_PHY_FSL_LYNX_28G=m

A Debian Bug has already been opened: https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1061117

As a temporary measure a patched version of Debian `6.5.0-0.deb12.4` is available [here](https://drive.google.com/file/d/1UTKVZW9vooaYw-bNgbJnCWAM4nGsOt1O/view?usp=sharing).
It is installed using the commands below:

```
sudo dpkg -i linux-image-6.5.0-0.deb12.4-arm64-unsigned_6.5.10-1~bpo12+1_arm64.deb
# success indicated by these messages:
flash-kernel: installing version 6.5.0-0.deb12.4-arm64
Generating boot script u-boot image... done.
Taking backup of boot.scr.
Installing new boot.scr.

# optionally to allow building custom kernel modules:
sudo apt-get install build-essential
sudo dpkg -i linux-kbuild-6.5.0-0.deb12.4_6.5.10-1~bpo12+1_arm64.deb linux-headers-6.5.0-0.deb12.4-common_6.5.10-1~bpo12+1_all.deb linux-headers-6.5.0-0.deb12.4-arm64_6.5.10-1~bpo12+1_arm64.deb
```

Once Debian integrated the changes, follow instructions below:

Upgrade Kernel via [backports](https://backports.debian.org/Instructions/), adding the missing drivers.

    apt-get install -t bookworm-backports linux-image-arm64

#### Install Restool

```
sudo apt-get install --no-install-recommends build-essential git pandoc
git clone https://github.com/nxp-qoriq/restool.git
cd restool
git reset --hard LSDK-21.08
make
sudo make install
```

#### Activate SFP Interfaces

Affected by [Debian Bug #1061117](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1061117)

Each SFP port is connected to a "dpmac" object managed by the network coprocessor.

To start all 4 SFPs, execute the commands below in order. They successively create new interfaces `eth1`, `eth2`, `eth3`, `eth4`:

    sudo ls-addni dpmac.7
    sudo ls-addni dpmac.8
    sudo ls-addni dpmac.9
    sudo ls-addni dpmac.10

### workaround for "task cryptomgr_test:493 blocked for more than 241 seconds" kernel log spam

Something is broken in Linux 6.5 for cryptomgr module tests, see e.g. [random form conversation about this issue](https://community.mnt.re/t/changes-introduced-by-kernel-6-5-rc7-1-exp1-reform20230831t205634z1/1651/10).

These tests can be disabled by adding kernel argument `cryptomgr.notests`:

1. Edit `/etc/default/flash-kernel`,
   update `LINUX_KERNEL_CMDLINE_DEFAULTS` variable to include `cryptomgr.notests`, e.g.:

   ```
   LINUX_KERNEL_CMDLINE="quiet"
   LINUX_KERNEL_CMDLINE_DEFAULTS="cryptomgr.notests"
   ```

2. Regenerate debian boot-script:

   ```
   sudo flash-kernel
   Using DTB: freescale/fsl-lx2160a-clearfog-cx.dtb
   Installing /etc/flash-kernel/dtbs/fsl-lx2160a-clearfog-cx.dtb into /boot/dtbs/6.5.0-0.deb12.4-arm64/freescale/fsl-lx2160a-clearfog-cx.dtb
   Taking backup of fsl-lx2160a-clearfog-cx.dtb.
   Installing new fsl-lx2160a-clearfog-cx.dtb.
   flash-kernel: installing version 6.5.0-0.deb12.4-arm64
   Generating boot script u-boot image... done.
   Taking backup of boot.scr.
   Installing new boot.scr.
   ```

### Install "amdgpu" Firmware

Note: Desktop-class GPUs by AMD and NVIDIA are not currently supported well on AArch64, success may vary.

Recent AMD graphics card using the modern "amdgpu" kernel driver require proprietary firmware.
The kernel may indicate this by the message below:

    sudo dmesg | grep amdgpu
    [    4.297767] [drm:amdgpu_pci_probe [amdgpu]] *ERROR* amdgpu requires firmware installed
    [    4.306038] amdgpu: See https://wiki.debian.org/Firmware for information about missing firmware

1. Add "non-free-firmware" component to `/etc/apt/sources.list`, e.g.:

       deb http://deb.debian.org/debian/ bookworm main non-free-firmware

       deb-src http://deb.debian.org/debian/ bookworm main non-free-firmware

       deb http://security.debian.org/debian-security bookworm-security main non-free-firmware
       deb-src http://security.debian.org/debian-security bookworm-security main non-free-firmware

       # bookworm-updates, to get updates before a point release is made;
       # see https://www.debian.org/doc/manuals/debian-reference/ch02.en.html#_updates_and_backports
       deb http://deb.debian.org/debian/ bookworm-updates main non-free-firmware
       deb-src http://deb.debian.org/debian/ bookworm-updates main non-free-firmware

2. Install "firmware-amd-graphics" package:

       sudo apt-get update
       sudo apt-get install firmware-amd-graphics

### Support large BARs for PCI-E

Some PCI devices such as machine-learning accelerators or graphics cards request large PCI bars.
The default configuration of Linux for LX2160A is limited to 1GB of 32-bit memory.

See below a typical kernel error message encountered with a Radeon Pro WX2100:

    [    1.002255] pci 0001:00:00.0: BAR 13: no space for [io  size 0x1000]
    [    1.002259] pci 0001:00:00.0: BAR 13: failed to assign [io  size 0x1000]

There is a [patch for device-tree posted as RFC](https://lore.kernel.org/r/20240429-lx2160-pci-v2-1-1b94576d6263@solid-run.com) to support 3GB of 32-bit memory, plus 16GB of 64-bit memory which can be applied manually by customizing device-tree files used by Debian:

2. Determine Kernel Version:

       uname -v
       #1 SMP Debian 6.1.85-1 (2024-04-11)

   Here Kernel is v6.1.85, for other versions substitute in the following steps accordingly.

2. Patch matching Kernel Sources and rebuild Device-Tree:

   **Note this step may be performed on any debian based linux pc or on lx2160a, whichever is convenient.**

       sudo apt-get update
       sudo apt-get install bison build-essential flex git

       git config --global user.name "<insert name>"
       git config --global user.email "<insert email address>"

       git clone https://kernel.googlesource.com/pub/scm/linux/kernel/git/stable/linux.git
       cd linux
       git checkout -b linux-6.1.y origin/linux-6.1.y
       git reset --hard v6.1.85

       wget -O bars.patch https://lore.kernel.org/all/20240429-lx2160-pci-v2-1-1b94576d6263@solid-run.com/raw
       git am -3 bars.patch

       make ARCH=arm64 defconfig
       make freescale/fsl-lx2160a-clearfog-cx.dtb
       make freescale/fsl-lx2160a-honeycomb.dtb

       # finally copy the patched DTBs to home directory
       cp -v arch/arm64/boot/dts/freescale/*.dtb ~/

3. Install patched DTB to target system:

   Copy patched DTBs to `/etc/flash-kernel/dtbs/` where `flash-kernel` command looks for custom DTBs
   which will be preferred over those shipped with the kernel package:

       sudo cp -v ~/*.dtb /etc/flash-kernel/dtbs/

   Regenerate boot-files:

       sudo flash-kernel

   When successfull the last command will log copying of patched DTB, e.g.:

       ...
       Installing /etc/flash-kernel/dtbs/fsl-lx2160a-honeycomb.dtb into /boot/dtbs/6.1.0-20-arm64/freescale/fsl-lx2160a-honeycomb.dtb
       ...

Reboot system to apply the changes.

### Avoid visual glitches with X11 and amdgpu

When using X11 with amd graphics cards there can be visual glitches and performance problems in default configuration.

Creating `/etc/drirc` with the settings below seems to provide good results:

```
<driconf>
    <!-- Please always enable app-specific workarounds for all drivers and
         screens. -->
    <device>
        <application name="XWayland" executable="Xwayland">
            <option name="mesa_extension_override" value="-GL_ARB_buffer_storage" />
        </application>
        <application name="Xorg" executable="Xorg">
            <option name="mesa_extension_override" value="-GL_ARB_buffer_storage" />
        </application>
    </device>
</driconf>
```
