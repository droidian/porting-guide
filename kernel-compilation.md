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

An example kernel packaged using this guide is the [Android Kernel for the F(x)tec Pro1](https://github.com/droidian-devices/linux-android-fxtec-pro1/tree/droidian/debian).

Prerequisites
-------------

* Device kernel sources
* A Halium-compliant kernel defconfig
* Docker

If you do not have a Halium-compliant kernel yet you should modify your kernel's configuration as suggested in [Kernel adaptation](#kernel-adaptation) later in the guide. Before doing this **make sure your kernel sources can be built and can boot Android**.

Package bring-up
----------------

Assuming your kernel sources are located in `~/droidian/kernel/vendor/device`,
you should create a new branch to house the Debian packaging.

We're also assuming that you want the resulting packages in `~/droidian/packages`.

Droidian tooling expects the kernel source to have a working git directory structure (be a kernel cloned from a git repository).

	(host)$ KERNEL_DIR="$HOME/droidian/kernel/vendor/device"
	(host)$ PACKAGES_DIR="$HOME/droidian/packages"
	(host)$ mkdir -p $PACKAGES_DIR
	(host)$ cd $KERNEL_DIR
	(host)$ git checkout -b droidian

Now it's time to fire up the Docker container.

	(host)$ docker run --rm -v $PACKAGES_DIR:/buildd -v $KERNEL_DIR:/buildd/sources -it quay.io/droidian/build-essential:current-amd64 bash

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

* `KERNEL_PREBUILT_DT` is available for devices with a prebuilt DT image (such as samsungs) and takes a path in the kernel tree.

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

* If your device requires a toolchain which is not included in the buildsystem, you can manually download the toolchain and add the path to `BUILD_PATH`.

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

### Toolchains

Droidian ships

* gcc 4.9 (legacy kernels)
* clang-android-6.0-4691093 (recommended toolchain for android9)
* clang-android-9.0-r353983c (recommended toolchain for android10)
* clang-android-10.0-r370808 (recommended toolchain for android11)
* clang-android-12.0-r416183b (recommended toolchain for android12-12.1)
* clang-android-14.0-r450784d (recommended toolchain for android13)

To use `clang-android-6.0-4691093` add it to `DEB_TOOLCHAIN` and set `BUILD_PATH` to the following value

`/usr/lib/llvm-android-6.0-4691093/bin`

To use `clang-android-9.0-r353983c` add it to `DEB_TOOLCHAIN` and set `BUILD_PATH` to the following value

`/usr/lib/llvm-android-9.0-r353983c/bin`

To use `clang-android-10.0-r370808` add it to `DEB_TOOLCHAIN` and set `BUILD_PATH` to the following value

`/usr/lib/llvm-android-10.0-r370808/bin`

To use `clang-android-12.0-r416183b` add it to `DEB_TOOLCHAIN` and set `BUILD_PATH` to the following value

`/usr/lib/llvm-android-12.0-r416183b/bin`

To use `clang-android-14.0-r450784d` add it to `DEB_TOOLCHAIN` and set `BUILD_PATH` to the following value

`/usr/lib/llvm-android-14.0-r450784d/bin`

In case you're on an older device and your kernel does not compile with any of the clang toolchains you can fallback to GCC

To use GCC change `BUILD_CC` to from `clang` to `aarch64-linux-android-gcc-4.9`

Keep in mind that any kernel newer than 4.4 should should compile fine with clang, there are always exceptions.

### Build target

Build target is passed to `make` which tells the make command what we expect at the end of the build.

Default build target is `Image.gz` which generates an image and adds the dtb image after making the gzip archive.

`Image.gz-dtb` can also be used for older devices that require the DTB appended to the end of the kernel.

`Image` is also available if your device requires a prebuilt dt

You can find which build target is required for your device by looking at the device tree of your device.

### initramfs hooks

In case of any issues with initramfs (as an example unl0kr showing a black screen while encryption is enabled), scripts can be added to the packaging which will be included in the ramdisk.

[This packaging](https://github.com/droidian-devices/linux-android-fxtec-pro1x/blob/droidian/debian/initramfs-overlay/scripts/halium-hooks) can be taken as a reference.

The following adds a function which will run in the initramfs on boot.

Any command that will aid the system in booting can be added to the script.

### build rules

On builds failing after the kernel compilation stage (during packaging), errors can be ignored.

build sequence can be altered with `override_dh_sequence` (make sure to replace sequence with your own sequence)

As as example

```
override_dh_dwz:
override_dh_strip:
override_dh_makeshlibs:
```

can be added to the end of `debian/rules`.

It is recommended to come back and fix the build errors later on instead of ignoring them as they might cause various issues in the future.

Kernel adaptation
-----------------

As a bare minimum these options need to be enabled in your defconfig

Some of these options might not be available to you depending on your kernel version, they can be safely ignored.

```
CONFIG_DEVTMPFS=y
CONFIG_VT=y
CONFIG_NAMESPACES=y
CONFIG_MODULES=y
CONFIG_DEVPTS_MULTIPLE_INSTANCES=y
CONFIG_USB_CONFIGFS_RNDIS=y
CONFIG_USB_CONFIGFS_RMNET_BAM=y
CONFIG_USB_CONFIGFS_MASS_STORAGE=y
CONFIG_INIT_STACK_ALL_ZERO=y
CONFIG_ANDROID_PARANOID_NETWORK=n
CONFIG_ANDROID_BINDERFS=n
```

Usually `CONFIG_NAMESPACES` enables all the namespace options but if it did not, all these options should be added

```
CONFIG_SYSVIPC=y
CONFIG_PID_NS=y
CONFIG_IPC_NS=y
CONFIG_UTS_NS=y
```

Later on for other components, various options should be enabled after the initial boot is done successfully.

For Bluetooth these options are required

```
CONFIG_BT=y
CONFIG_BT_HIDP=y
CONFIG_BT_RFCOMM=y
CONFIG_BT_RFCOMM_TTY=y
CONFIG_BT_BNEP=y
CONFIG_BT_BNEP_MC_FILTER=y
CONFIG_BT_BNEP_PROTO_FILTER=y
CONFIG_BT_HCIVHCI=y
```

For Waydroid

```
CONFIG_SW_SYNC_USER=y
CONFIG_NET_CLS_CGROUP=y
CONFIG_CGROUP_NET_CLASSID=y
CONFIG_VETH=y
CONFIG_NETFILTER_XT_TARGET_CHECKSUM=y
CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder,anbox-binder,anbox-hwbinder,anbox-vndbinder"
```

For Plymouth (boot animation)

```
# CONFIG_FB_SYS_FILLRECT is not set
# CONFIG_FB_SYS_COPYAREA is not set
# CONFIG_FB_SYS_IMAGEBLIT is not set
# CONFIG_FB_SYS_FOPS is not set
# CONFIG_FB_VIRTUAL is not set
```

To ease debugging, pstore can be enabled to get logs on each boot

```
CONFIG_PSTORE=y
CONFIG_PSTORE_CONSOLE=y
CONFIG_PSTORE_RAM=y
CONFIG_PSTORE_RAM_ANNOTATION_APPEND=y
```

If you have pstore enabled, you might find clues in `/sys/fs/pstore` from your recovery.

You can use menuconfig to make sure all the options are enabled with all their dependencies.

	(docker)# mkdir -p out/KERNEL_OBJ && make ARCH=arm64 O=out/KERNEL_OBJ/ your_defconfig && make ARCH=arm64 O=out/KERNEL_OBJ/ menuconfig

After modifying your defconfig, copy `out/KERNEL_OBJ/.config` to `arch/YOURARCH/configs/your_defconfig`.

As an alternative, `KERNEL_CONFIG_USE_FRAGMENTS = 1` can be set in `kernel-info.mk` to include defconfig fragments inside your defconfig on build time.

defconfig fragments should be placed in the root of the kernel source in a directory called droidian. [this source](https://github.com/droidian-devices/linux-android-fxtec-pro1x/tree/droidian/droidian) can be taken as a reference.

`KERNEL_CONFIG_USE_DIFFCONFIG` can be enabled to use the python script `diffconfig` to compare fragments and the main defconfig.

to use diffconfig a diff file should be provided like so

`KERNEL_PRODUCT_DIFFCONFIG = diff_file`

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

DTB/DTBO compilation failure
----------------------------

In case of DTB/DTBO compilation failure it is a good idea to try using an external dtc compiler instead of the one presend in the kernel tree.

To do so after building the control file, add `device-tree-compiler` to Build-Depends then and in `debian/rules` after the include line add

`BUILD_COMMAND := $(BUILD_COMMAND) DTC_EXT=/usr/bin/dtc`

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

...and then push your `droidian` branch for others to enjoy.
