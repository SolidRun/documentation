## Introduction

This page describes how to build, install and update U-Boot for SolidRun Armada 8040-based devices.

## Prebuilt Binaries

Usable firmware binaries are available at https://images.solid-run.com/8040/U-Boot.

## Build Instructions

### Fetch sources

    git clone http://git.denx.de/u-boot.git
    git clone https://github.com/MarvellEmbeddedProcessors/atf-marvell.git -b atf-v1.5-armada-18.12
    git clone https://github.com/MarvellEmbeddedProcessors/mv-ddr-marvell.git -b mv_ddr-armada-18.12
    git clone https://github.com/MarvellEmbeddedProcessors/binaries-marvell.git -b binaries-marvell-armada-18.12
    cd u-boot; git checkout v2019.04

For U-Boot itself mainline is fully usable – both the latest stable release, and master (for the adventurous people) will do. The latest Marvell release at the time of writing was 18.12; newer versions are usually safe to use.

### Compile

A cross toolchain targeting arm64 is required. Linaro provides prebuilt toolchains [here](https://www.linaro.org/downloads/); Generally though any toolchain will do. On Debian Stretch and later, the arm64 toolchain is provided by the crossbuild-essential-arm64 package.

    export CROSS_COMPILE=[path-to-your-toolchain/]aarch64-linux-gnu-

    # 1. U-Boot
    cd u-boot
    make mvebu_mcbin-88f8040_defconfig
    # run make menuconfig to customize (optional)
    # for the Clearfog GT, DEFAULT_DEVICE_TREE needs setting to "armada-8040-clearfog-gt-8k"
    # for the Singleshot, DEFAULT_DEVICE_TREE needs setting to "armada-8040-mcbin-singleshot"
    # --> Device Tree Control --> Default Device Tree for DT control
    make -j4

    # 2. ATF
    cd atf-marvell
    make PLAT=a80x0_mcbin MV_DDR_PATH=../mv-ddr-marvell SCP_BL2=/dev/null clean
    make PLAT=a80x0_mcbin MV_DDR_PATH=../mv-ddr-marvell SCP_BL2=../binaries-marvell/mrvl_scp_bl2.img BL33=../u-boot/u-boot.bin all fip

If everything went well there should now be a new file at `atf-marvell/build/a80x0_mcbin/release/flash-image.bin` ready for deployment.

### Configure

The above flash-image.bin can be used to boot from Micro-SD, SPI, eMMC and through UART with xmodem.
Please note however that the environment will always be saved on SPI flash unless it was explicitly configured differently through these configuration items:

    # SPI Flash
    CONFIG_ENV_IS_IN_MMC=n
    CONFIG_ENV_IS_IN_SPI_FLASH=y

    # microSD
    CONFIG_ENV_IS_IN_MMC=y
    CONFIG_SYS_MMC_ENV_DEV=1
    CONFIG_SYS_MMC_ENV_PART=0
    CONFIG_ENV_IS_IN_SPI_FLASH=n

    # eMMC boot0
    CONFIG_ENV_IS_IN_MMC=y
    CONFIG_SYS_MMC_ENV_DEV=0
    CONFIG_SYS_MMC_ENV_PART=1
    CONFIG_ENV_IS_IN_SPI_FLASH=n

    # eMMC boot1
    CONFIG_ENV_IS_IN_MMC=y
    CONFIG_SYS_MMC_ENV_DEV=0
    CONFIG_SYS_MMC_ENV_PART=2
    CONFIG_ENV_IS_IN_SPI_FLASH=n

Note: Since not all of the options are exposed by menuconfig, appending them to `configs/mvebu_mcbin-88f8040_defconfig` **before** running `make mvebu_mcbin-88f8040_defconfig` is the easiest.

## Deploy

### From Linux

#### to microSD

First insert the target microSD into any computer running Linux and identify its canonical name in /dev, e.g. by reading through dmesg. Errors in this step **will** result in data loss!

The Boot ROM expects to find a bootable image at 512 bytes into the block device. Use dd for writing the previously compiled flash-image.bin to the designated location. In this example sdX is used as placeholder for the actual device name of your microSD on your system:

    dd if=flash-image.bin of=/dev/sdX bs=512 seek=1 conv=sync

This process will also work on the device itself, if it has already booted into Linux.

#### to eMMC data partition

Since the eMMC is soldered to the board, this procedure has to be done on the device itself after booting into a Linux system first. The process is identical to microSD except for the important detail that the Boot ROM expects to find the bootable image at the first block. This **will** conflict with any partition table or filesystem on this partition. Therefore using one of the dedicated boot partitions is recommended.

    dd if=flash-image.bin of=/dev/sdX conv=sync

#### to eMMC bootY

Since the eMMC is soldered to the board, this procedure has to be done on the device itself after booting into a Linux system first. Please note that as with the eMMC data partition, the Boot ROM expects to find the bootable image at the start of the partition without any offset.

To avoid accidents, the boot partitions are write protected by default. This protection is easy enough to turn off:

    # for boot0:
    echo 0 | sudo tee /sys/block/mmcblk0boot0/force_ro

    # for boot1:
    echo 0 | sudo tee /sys/block/mmcblk0boot1/force_ro

Now the bootable image can be written to either boot partition with dd:

    # for boot0
    sudo dd if=flash-image.bin of=/dev/mmcblk0boot0 conv=fsync

    # for boot1
    sudo dd if=flash-image.bin of=/dev/mmcblk0boot1 conv=fsync

As a last step, the eMMC has to be configured for selecting the intended boot partition. This can be done on the U-Boot console with the `mmc partconf` command, or from Linux with the `mmc` application from mmc-utils:

    # from Linux:
    # use boot0
    mmc bootpart enable 1 0 /dev/mmcblk0
    # use boot1
    mmc bootpart enable 2 0 /dev/mmcblk0

    # from U-Boot
    # use boot0
    mmc partconf 0 0 1 0
    # use boot1
    mmc partconf 0 0 2 0

### From U-Boot

#### to SPI Flash

This step requires U-Boot running on the target device first, e.g. loaded from microSD or UART. The flash-image.bin can then be loaded from either microSD, eMMC, USB or network, and finally written to the SPI flash.
This sample covers the easiest case where flash-image.bin is available on a fat formatted partition on a USB drive:

    # start USB stack
    usb start
    # scanning usb for storage devices... 1 Storage Device(s) found <-- indicates success

    # load flash-image.bin to ram
    load usb 0:1 $kernel_addr_r flash-image.bin
    # u-boot will indicate how many btes were read. Make sure to verify the number!

    # initialize spi flash
    sf probe

    # erase
    sf erase 0 0x800000

    # finally write loaded file
    sf write $kernel_addr_r 0 0x$filesize

## Booting the board using UART xmodem

The Armada 8040 can be booted through UART xmodem. Even if the processor is indicated to boot from SPI, Micro SD or eMMC the bootrom inside the processor first checks if there is a pattern on it’s UART RX and decides if to continue booting from the designated boot sources, or use UART for that.

This is mostly used for system manufacturing, unbricking etc…

The flash-image.bin that is built above is a ready to use xmodem protocol transferable image to Armada 8040 processor. But first the processor needs to be redirected from it’s default boot vector.

This can be achieved by running the ‘download-serial.sh’ script that is part of [SolidRun Armada 388 U-Boot](https://github.com/SolidRun/u-boot-armada38x/blob/u-boot-2013.01-15t1-clearfog/download-serial.sh)

An example is as follows:

    ./download-serial.sh /dev/ttyUSB0 flash-image.bin

Embedded in `download-serial.sh` is a small C program that gets built every time the script runs and requires curses libraries.
