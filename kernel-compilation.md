Kernel compilation
==================

The stock Android kernel is unfortunately not enough to be able to run Droidian.

The good news is that, on GSI-capable devices, often only kernel changes are necessary.

Table of contents
-----------------

* [Summary](#summary)
* [Prerequisites](#prerequisites)
* [Package bring-up](#package-bring-up)
* [Kernel info options](#kernel-info-options)
* [Kernel adaptation](#kernel-adaptation)
* [Compiling](#compiling)
* [Obtaining the boot image](#obtaining-the-boot-image)
* [Committing changes](#committing-changes)

Summary
-------

Droidian runs on [halium](https://halium.org).
If your device is launched with Android 9 or above it is possible to port Droidian to it.
If it already has a halium-compliant kernel of halium-9.0 and above chances are that Droidian will work without much modification.

If your device has shipped with Android 8.1 or 9, it probably is GSI (Generic System Image) capable, and such it's possible to use an already available, generic Android System Image with Halium patches applied.
This will reduce the amount of work the porter has to do significantly.

If your device doesn't support GSI, you'll also need to compile a patched system image.
The documentation for image creation can be found in [this guide](./image-creation.md)

On Halium, the Android kernel is built via the standard Android toolchain.
While this makes sense, on GSI-capable devices this can be a waste of time since often only kernel changes are required.

Thus, Droidian uses a different approach to compile kernels - the upside is that you get packaging for free so that kernels can be upgraded over-the-air via APT, if you wish so.

Note that this guide assumes that you're going to cross-compile an arm64
Android kernel on an x86_64 (amd64) machine using the Android-supplied
precompiled toolchain that's available in the Droidian repositories.
It's trivial to disable cross-compiling and compiling using the standard
Debian toolchain.

Using this method you can also compile and package mainline kernels.

An example kernel packaged using this guide is the [Android Kernel for the F(x)tec Pro1](https://github.com/droidian-devices/linux-android-fxtec-pro1/tree/bookworm/debian).

Prerequisites
-------------

* Device kernel sources
* A Halium-compliant kernel defconfig
* Docker

If you do not have a Halium-compliant kernel yet you should modify your kernel's configuration as suggested in [Kernel adaptation](#kernel-adaptation) later in the guide.

Package bring-up
----------------

Assuming your kernel sources are located in `~/droidian/kernel/vendor/device`,
you should create a new branch to house the Debian packaging.

We're also assuming that you want the resulting packages in `~/droidian/packages`.

Droidian tooling expects the kernel source to have a working git directory structure (be a kernel cloned from a git repository) and the branch to be named after the Debian codename, such as `bookworm`.

	(host)$ KERNEL_DIR="$HOME/droidian/kernel/vendor/device"
	(host)$ PACKAGES_DIR="$HOME/droidian/packages"
	(host)$ mkdir -p $PACKAGES_DIR
	(host)$ cd $KERNEL_DIR
	(host)$ git checkout -b bookworm

Now it's time to fire up the Docker container.

	(host)$ docker run --rm -v $PACKAGES_DIR:/buildd -v $KERNEL_DIR:/buildd/sources -it quay.io/droidian/build-essential:bookworm-amd64 bash

Inside the Docker container, install the `linux-packaging-snippets`, that
provides the example `kernel-info.mk` file.

	(docker)# apt-get install linux-packaging-snippets

And create the skeleton packaging:

	(docker)# cd /buildd/sources
	(docker)# mkdir -p debian/source
	(docker)# cp -v /usr/share/linux-packaging-snippets/kernel-info.mk.example debian/kernel-info.mk
	(docker)# echo 13 > debian/compat
	(docker)# echo "3.0 (native)" > debian/source/format
	(docker)# cat > debian/rules <<EOF
	#!/usr/bin/make -f
	
	include /usr/share/linux-packaging-snippets/kernel-snippet.mk
	
	%:
		dh \$@
	EOF
	(docker)# chmod +x debian/rules

Now edit `debian/kernel-info.mk` to match your kernel settings.

Most of the defaults are enough for building Android kernels with the Pie
toolchain, so you'll probably need to change device-specific settings (such
as vendor, name, cmdline, defconfig and the various offsets).

by unpacking an already built `boot.img` using unpackbootimg all the offsets can be found.

Kernel info options
-------------------

unpackbootimg syntax goes as follows

	(docker)# unpackbootimg --boot_img boot.img

or for the AOSP version of unpackbootimg

	(docker)# unpackbootimg -i boot.img

### kernel-info.mk entries

* `KERNEL_BASE_VERSION` is the kernel version which can be viewed in Makefile at the root of your kernel source.

As an example

```
VERSION = 4
PATCHLEVEL = 14
SUBLEVEL = 221
```

will be 4.14.221

* `KERNEL_DEFCONFIG` is the defconfig filename found at arch/YOURARCH/configs

* `KERNEL_IMAGE_WITH_DTB` determines whether or not to include a dtb file in the kernel. if this option is set `KERNEL_IMAGE_DTB` also needs to be set. if not an attempt to find it will occur.

* `KERNEL_IMAGE_DTB` is the path to the dtb file which can be found in arch/YOURARCH/boot/dts/SOC/

* `KERNEL_IMAGE_WITH_DTB_OVERLAY` determines whether or not to build a dtbo file. if this option is set `KERNEL_IMAGE_DTB_OVERLAY` also needs to be set. if not an attempt to find it will occur.

* `KERNEL_IMAGE_DTB_OVERLAY` is the path to the dtbo file which can be found in arch/YOURARCH/boot/dts/SOC/

All these values can be viewed by extracting a boot image with unpackbootimg

* `KERNEL_BOOTIMAGE_CMDLINE` corresponds to "command line args" or "BOARD_KERNEL_CMDLINE" `console=tty0` and `droidian.lvm.prefer` should be appended to the cmdline. Make sure to remove any `systempart` entry from cmdline.

* `KERNEL_BOOTIMAGE_PAGE_SIZE` corresponds to "page size" or "BOARD_PAGE_SIZE"

* `KERNEL_BOOTIMAGE_BASE_OFFSET` corresponds to "base" or "BOARD_KERNEL_BASE"

* `KERNEL_BOOTIMAGE_KERNEL_OFFSET` corresponds to "kernel load address" or "BOARD_KERNEL_OFFSET"

* `KERNEL_BOOTIMAGE_INITRAMFS_OFFSET` corresponds to "ramdisk load address" or "BOARD_RAMDISK_OFFSET"

* `KERNEL_BOOTIMAGE_SECONDIMAGE_OFFSET` corresponds to "second bootloader load address" or "BOARD_SECOND_OFFSET"

* `KERNEL_BOOTIMAGE_TAGS_OFFSET` corresponds to "kernel tags load address" or "BOARD_TAGS_OFFSET"

* `KERNEL_BOOTIMAGE_DTB_OFFSET` corresponds to "dtb address" or "BOARD_DTB_OFFSET"

Although this option is only required for kernel header version 2. it can be commented otherwise.

* `KERNEL_BOOTIMAGE_VERSION` correlates to the kernel header version. Devices launched with Android 8 and lower are 0, Android 9 is 1, Android 10 is 2 and Android 11 is 2 and GKI devices are 3.

* For Samsung devices `DEVICE_VBMETA_IS_SAMSUNG` must be set to 1.

* `BUILD_CC` for most devices launched with Android 9 and above is clang but if your kernel fails to build with `clang` you might try changing the value to `aarch64-linux-android-gcc-4.9` to build with gcc.

* `DEB_BUILD_FOR` and `KERNEL_ARCH` should be changed according to device architecture.

### Enabling automatic boot partition flashing

Installing the built packages means that the kernel and its modules are
put in place - but the user should still flash it to their boot partition.

If you wish to enable automatic flashing (via [flash-bootimage](https://github.com/droidian/flash-bootimage)),
you can do so by setting `FLASH_ENABLED` to 1 (which is the default).

If your device doesn't support A/B updates, be sure to set `FLASH_IS_LEGACY_DEVICE` to 1.

Note that you need to specify some device info so that `flash-bootimage` can cross-check when running on-device:

* `FLASH_INFO_MANUFACTURER`: the value of the `ro.product.vendor.manufacturer`
Android property. On a running Droidian system, you can obtain it with

	(device)$ sudo android_getprop ro.product.vendor.manufacturer

* `FLASH_INFO_MODEL`: the value of the `ro.product.vendor.model`
Android property. On a running Droidian system, you can obtain it with

	(device)$ sudo android_getprop ro.product.vendor.model

* `FLASH_INFO_CPU`: a relevant bit of info from `/proc/cpuinfo`.

If `FLASH_INFO_MANUFACTURER` or `FLASH_INFO_MODEL` are not defined (they both
are required for checking against the Android properties), `flash-bootimage`
will check for `FLASH_INFO_CPU`.

If no device-specific information has been specified, the kernel upgrade will fail.

An example for the F(x)tec Pro1:

```
FLASH_ENABLED = 1
FLASH_INFO_MANUFACTURER = Fxtec
FLASH_INFO_MODEL = QX1000
FLASH_INFO_CPU = Qualcomm Technologies, Inc MSM8998
```

Kernel adaptation
-----------------

As a bare minimum these options need to be enabled in your defconfig

```
CONFIG_DEVTMPFS
CONFIG_VT
CONFIG_NAMESPACES
CONFIG_MODULES
CONFIG_DEVPTS_MULTIPLE_INSTANCES
CONFIG_USB_CONFIGFS_RNDIS
```

Usually `CONFIG_NAMESPACES` enables all the namespace options but if it did not, all these options should be added

```
CONFIG_SYSVIPC
CONFIG_PID_NS
CONFIG_IPC_NS
CONFIG_UTS_NS
```

Later on for other components, various options should be enabled after the initial boot is done successfully.

As an example for Bluetooth these options might be required

```
CONFIG_BT
CONFIG_BT_HCIVHCI
```

You can use menuconfig to make sure all the options are enabled with all their dependencies.

	(docker)# mkdir -p out/KERNEL_OBJ && make ARCH=arm64 O=out/KERNEL_OBJ/ your_defconfig && make ARCH=arm64 O=out/KERNEL_OBJ/ menuconfig

After modifying your defconfig, copy `out/KERNEL_OBJ/.config` to `arch/YOURARCH/configs/your_defconfig`.

If LXC fails to start you can use `lxc-checkconfig` after device first booted to check for other options that might be needed.

Compiling
---------

Now that `kernel-info.mk` has been modified, the only thing that remains
is to actually compile the kernel.

First of all, (re)create the `debian/control` file:

	(docker)# rm -f debian/control
	(docker)# debian/rules debian/control
	
Now that everything is in place, you can start a build with `releng-build-package`:

	(docker)# RELENG_HOST_ARCH="arm64" releng-build-package
	
The `RELENG_HOST_ARCH` variable is required when cross-building.

If everything goes well, you'll find the resulting packages in `$PACKAGES_DIR`.

Obtaining the boot image
------------------------

The boot image is shipped into the `linux-bootimage-VERSION-VENDOR-DEVICE` package.

You can pick up the boot.img by extracting the package with `dpkg-deb` or
by picking up directly from the compiled artifacts (`out/KERNEL_OBJ/boot.img`).

The kernel image already embeds the Droidian initramfs.

Make sure to save all your `linux-*.deb` packages as you'll need those further in the guide.

Committing changes
------------------

When you're happy with the kernel, be sure to commit your changes as well as the debian packaging to a git repository:

* debian/source/
* debian/control
* debian/rules
* debian/compat
* debian/kernel-info.mk

...and then push your `bookworm` branch for others to enjoy.
