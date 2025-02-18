# SolidRun Debian 11 for i.MX6 - Release 2

## Summary

- Upstream Release: Debian 12
- SolidRun Release: 1
- Hardware
  - ~~Cubox-i~~ (not tested)
  - HummingBoard Base
  - HummingBoard Pro
  - HummingBoard Edge
  - HummingBoard Gate
  - SoCs:
    - i.MX6 Solo
    - i.MX6 DualLite
    - i.MX6 Quad
- Features:
  - Ports:
    - microSD
    - eMMC
    - mSATA (HummingBoard only)
    - ~~eSATA (Cubox-i only)~~ (not tested)
    - Ethernet
    - SoM integrated WiFi+Bluetooth
    - 2x USB-2.0 Type A
    - additional 2x USB-2.0 Type A (HummingBoard Edge/Gate only)
    - ~~SPDIF coax (HummingBoard Base+Pro only)~~ (not tested)
    - ~~SPDIF optical (some Cubox-i only)~~ (not tested)
    - ~~3.5mm audio jack (HummingBoard only)~~ (not tested)
    - HDMI
    - ~~MIPI-CSI-2~~ (not tested)
  - Multimedia:
    ~~- OpenCL~~ (not currently supported)
    - OpenGL-ES 2.0
    - GStreamer HW-Accelerated Video Decoder
      - MPEG2
      - ~~MPEG4~~ (not tested)
      - H.264
    - ALSA Audio Playback
      - ~~HDMI~~ (not tested)
      - ~~SPDIF coax~~ (not tested)
      - ~~SPDIF optical~~ (not tested)
      - ~~3.5mm jack: analog stereo (HummingBoard Pro+Gate+Edge only)~~ (not tested)
- Major Components:
  - U-Boot v2018.01 SolidRun Fork
  - Linux 6.1 from the Debian project

## Download

- [i.MX6, microSD bootable](https://images.solid-run.com/IMX6/Debian/sr-imx6-debian-bookworm-20230809-cli-sdhc.img.xz)

# Documentation

## Install

All images are designed for raw deployment byte by byte (**not as a file**) to block storage such as microSD cards. Several tools can be used, including `dd`, `win32diskimager` and [etcher.io](https://etcher.io/). Note that care must be taken to decompress the image file for the first 2, while *etcher.io* can also handle the compressed image.

## Log In

Once greeted by a login prompt, the default username and password are both "debian". For security reasons there is **no** root password! If you really need one, you can run `sudo passwd root` to set your own.
For remote sessions, ssh is preconfigured. Check your router for finding the devcies IP address, or try the default hostname *sr-imx6*.

## Usage Hints

### Install to eMMC

If you have the optional eMMC on your SOM use these instructions to install Debian on the eMMC and boot from there.

1. Set the [boot select jumpers](https://solidrun.atlassian.net/wiki/spaces/developer/pages/286621835/HummingBoard+Edge+Gate+Boot+Jumpers) to SD card

2. Boot from SD

3. Download the Debian image

4. As root write the downloaded image to the eMMC:

       xz -dc sr-imx6-debian-bullseye-20230807-cli-sdhc.img.xz | dd of=/dev/mmcblk2 bs=4M conv=fsync

5. Find bootloader images:

  Binaries of the SolidRun u-boot fork are available for download:

      wget -O SPL https://images.solid-run.com/IMX6/U-Boot/spl-imx6-sdhc.bin
      wget -O u-boot.img https://images.solid-run.com/IMX6/U-Boot/u-boot-imx6-sdhc.img

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

### Wayland

While it is possible to install full destop environmnts such as Gnome, for testing functionality the reference compositor weston should be used.
However note that weston refuses to start from an interactive commandline session.
Instead wayland sessions must be initiated from a session manager such as sddm, gdm3 or lightdm.

    # install session-manager and weston
    sudo apt-get install lightdm weston
    systemctl start lightdm

Log-In through the graphical login prompt, choosing session type "Weston" under the small wrench icon in the upper right corner.
By default only one application is available, the terminal emulator, at the upper left corner.vl

### X11

Note X11 is legacy software and may experience artifacts and glitches that may not be fixed.

Install:

    sudo apt install twm xinit xserver-xorg xserver-xorg-input-evdev xterm

Run:

    # start X server FROM A PHYSICAL TERMINAL not remote or serial session
    startx

### OpenGL-ES

- eglinfo

  A Mesa demo application for showing supported EGL APIs and extensions

      sudo apt-get install mesa-utils-extra

      eglinfo

- glmark2

  Benchmark for OpenGL-ES 2.0 - available [here on github](https://github.com/glmark2/glmark2).
  Debian repositories provide suitable binaries that can be installed through apt:

      sudo apt-get install glmark2-es2-drm glmark2-es2-wayland glmark2-es2-x11

      # from text-mode session
      glmark2-es2-drm
      # from a wayland session
      glmark2-es2-wayland
      # from an X session
      glmark2-es2

### GStreamer Examples

Suggested packages for Audio and Video:

    sudo apt-get install gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-alsa gstreamer1.0-tools

- Text-Mode (KMS):

      export GST_GL_PLATFORM=egl GST_GL_API=gles2 GST_GL_WINDOW=gbm

      wget https://download.blender.org/peach/bigbuckbunny_movies/big_buck_bunny_1080p_h264.mov
      # H.264: video only
      gst-launch-1.0 filesrc location=$PWD/big_buck_bunny_1080p_h264.mov ! parsebin ! v4l2h264dec ! videoconvert ! kmssink

      wget http://docs.evostream.com/sample_content/assets/bun33s.ts
      # MPEG2: video only
      gst-launch-1.0 filesrc location=$PWD/bun33s.ts ! parsebin ! v4l2mpeg2dec ! videoconvert ! kmssink

- Wayland:

      export GST_GL_PLATFORM=egl GST_GL_API=gles2 GST_GL_WINDOW=wayland

      wget https://download.blender.org/peach/bigbuckbunny_movies/big_buck_bunny_1080p_h264.mov
      # H.264: video only
      gst-launch-1.0 filesrc location=$PWD/big_buck_bunny_1080p_h264.mov ! parsebin ! v4l2h264dec ! videoconvert ! waylandsink fullscreen=1
      # H.264, AAC: video + analog stereo audio (3.5mm jack)
      gst-launch-1.0 filesrc location=$PWD/big_buck_bunny_1080p_h264.mov ! parsebin name=p p. ! queue ! v4l2h264dec ! videoconvert ! waylandsink fullscreen=1 p. ! queue ! avdec_aac ! audioconvert ! alsasink device=dmix:CARD=Codec

      wget http://docs.evostream.com/sample_content/assets/bun33s.ts
      # MPEG2: video only
      gst-launch-1.0 filesrc location=$PWD/bun33s.ts ! parsebin ! v4l2mpeg2dec ! videoconvert ! waylandsink

- X11:

      export GST_GL_PLATFORM=egl GST_GL_API=gles2 GST_GL_WINDOW=x11

      wget https://download.blender.org/peach/bigbuckbunny_movies/big_buck_bunny_1080p_h264.mov
      # H.264: video only
      gst-launch-1.0 filesrc location=$PWD/big_buck_bunny_1080p_h264.mov ! parsebin ! v4l2h264dec ! videoconvert ! xvimagesink

      wget http://docs.evostream.com/sample_content/assets/bun33s.ts
      # MPEG2: video only
      gst-launch-1.0 filesrc location=$PWD/bun33s.ts ! parsebin ! v4l2mpeg2dec ! videoconvert ! xvimagesink

### MIPI-CSI2 Video Capture

TBD

### ALSA Examples

    sudo apt-get install vorbis-tools
    wget https://github.com/KDE/amarok/raw/master/data/first_run_jingle.ogg
    # play to 3.5mm jack
    ogg123 -d alsa -o dev:dmix:CARD=Codec first_run_jingle.ogg
    # play to hdmi
    ogg123 -d alsa -o dev:dmix:CARD=DWHDMI first_run_jingle.ogg
    # play to spdif
    ogg123 -d alsa -o dev:dmix:CARD=SPDIF first_run_jingle.ogg

### HummingBoard CBi

#### Initial Setup

HummingBoard CBi can use the CAN-Bus and RS485 only if booted with the appropriate device-tree.
From U-Boot console, interrupt boot by pressing any key during the `Hit any key to stop autoboot` prompt.
Then permanently choose the cbi device-tree variant:

    Hit any key to stop autoboot:  0
    => setenv is_cbi yes
    => saveenv
    => reset

From now on this system is permanently configured as a HummingBoard CBi with i.MX6 DualLite SoM rev. 1.5 or later, and eMMC.
Note that other SoMs need to change the fdtfile accordingly.

Note: U-Boot environment not saving!!!

#### CAN-Bus

After booting with the appropriate device-tree, the CAN network interface will show up in the output of the `ip` utility:

    debian@sr-imx6:~$ ip addr
    ...
    3: can0: <NOARP,ECHO> mtu 16 qdisc noop state DOWN group default qlen 10
        link/can

For transmitting packets it is required to configure a bitrate and set link state up.
Then data can be exchanged e.g. with the `candump` and `cansend` commands from `can-utils` package:

    sudo apt-get install can-utils
    sudo ip link set can0 up type can bitrate 125000
    sudo ip link set dev can0 up

    # To receive data
    candump can0

    # To send data
    cansend can0 "123#c0ffee"

#### RS485

RS485 does not auto-negotiate.
Instead demo application is available on GitHub for configuring connection properties.
On Debian it can be installed as follows:

    sudo apt-get install --no-install-recommends gcc git libc6-dev linux-libc-dev make
    git clone https://github.com/mniestroj/rs485conf.git
    cd rs485conf
    make
    sudo make install

Then to send a data blob between two devices:

    # On the receiving end:
    sudo rs485conf -e 1 /dev/ttymxc1
    sudo cat /dev/ttymxc1

    # On the sending end:
    printf "Hallo\n" | sudo tee /dev/ttymxc1

Note that `ttymxc1` is the RS485 device on HummingBoard CBi only, other devices will differ.

## Known Issues

### No IP on eth0 when connecting cable after boot

When booting the system without ethernet cable connected, the interface will not acquire an IP address by DHCP when hotplugging the cable later. As a workaround the interface can be cycled down and up:

    ifdown eth0
    ifup eth0
