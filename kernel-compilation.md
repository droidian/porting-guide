Kernel compilation
==================

The stock Android kernel is unfortunately not enough to be able to run
Droidian.

The good news is that, on GSI-capable devices, often only kernel changes
are necessary.

Table of contents
-----------------

* [Summary](#summary)
* [Prerequisites](#prerequisites)
* [Package bring-up](#package-bring-up)
* [Compiling](#compiling)
* [Obtaining the boot image](#obtaining-the-boot-image)
* [Committing changes](#committing-changes)

Summary
-------

Droidian runs on [halium](https://halium.org). If your device is
supported by halium-9.0, chances are that Droidian would work
on it.

If your device has shipped with Android 8.1 or 9, it probably is
GSI (Generic System Image) capable, and such it's possible to use
an already available, generic Android System Image with Halium patches
applied.

If your device doesn't support GSI, you'll also need to compile a patched
system image. This is beyond the scope of this document.

On Halium, the Android kernel is built via the standard Android toolchain.  
While this makes sense, on GSI-capable devices this can be a waste of time
since often only kernel changes are required.

Thus, Droidian uses a different approach to compile kernels - the
upside is that you get packaging for free so that kernels can be upgraded
over-the-air via APT, if you wish so.

Note that this guide assumes that you're going to cross-compile an arm64
Android kernel on an x86_64 (amd64) machine using the Android-supplied
precompiled toolchain that's available in the Droidian repositories.  
It's trivial to disable cross-compiling and compiling using the standard
Debian toolchain.

Using this method you can also compile and package mainline kernels.

An example kernel packaged using this guide is the [Android Kernel for the F(x)tec Pro1](https://github.com/droidian/linux-android-fxtec-pro1/tree/feature/bookworm/initial-packaging/debian).

Prerequisites
-------------

* Device kernel sources
* An Halium-compilant kernel defconfig
* Docker

The standard [Halium kernel compilation guide](http://docs.halium.org/en/latest/porting/build-sources.html#modify-the-kernel-configuration)
still applies. Please go through that paragraph and modify the kernel defconfig
as suggeested.

Package bring-up
----------------

Assuming your kernel sources are located in `~/droidian/kernel/vendor/device`,
you should create a new branch to house the Debian packaging.

We're also assuming that you want the resulting packages in `~/droidian/packages`.

Droidian tooling expects the branch to be named after the Debian
codename, such as `bookworm`

	(host)$ KERNEL_DIR="$HOME/droidian/kernel/vendor/device"
	(host)$ PACKAGES_DIR="$HOME/droidian/packages"
	(host)$ mkdir -p $PACKAGES_DIR
	(host)$ cd $KERNEL_DIR
	(host)$ git checkout -b bookworm

Now it's time to fire up the Docker container.

	(host)$ docker run -v $PACKAGES_DIR:/buildd -v $KERNEL_DIR:/buildd/sources -it quay.io/droidian/build-essential:bookworm-amd64 bash

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

You can find offsets by looking at the Android device tree, or by inspecting
an already built `boot.img`.

### Enabling automatic boot partition flashing

Installing the built packages means that the kernel and its modules are
put in place - but the user should still flash it to their boot partition.

If you wish to enable automatic flashing (via [flash-bootimage](https://github.com/droidian/flash-bootimage)),
you can do so by setting `FLASH_ENABLED` to 1 (which is the default).

If your device doesn't support A/B updates, be sure to set `FLASH_IS_LEGACY_DEVICE`
to 1.

Note that you need to specify some device info so that `flash-bootimage`
can cross-check when running on-device:

* `FLASH_INFO_MANUFACTURER`: the value of the `ro.product.vendor.manufacturer`
Android property. On a running Droidian system, you can obtain it with

```
sudo android_getprop ro.product.vendor.manufacturer
```

* `FLASH_INFO_MODEL`: the value of the `ro.product.vendor.model`
Android property. On a running Droidian system, you can obtain it with

```
sudo android_getprop ro.product.vendor.model
```

* `FLASH_INFO_CPU`: a relevant bit of info from `/proc/cpuinfo`.

If `FLASH_INFO_MANUFACTURER` or `FLASH_INFO_MODEL` are not defined (they both
are required for checking against the Android properties), `flash-bootimage`
will check for `FLASH_INFO_CPU`.

If no device-specific information has been specified, the kernel upgrade
will fail.

An example for the F(x)tec Pro1:

```
FLASH_ENABLED = 1
FLASH_INFO_MANUFACTURER = Fxtec
FLASH_INFO_MODEL = QX1000
FLASH_INFO_CPU = Qualcomm Technologies, Inc MSM8998
```

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

The boot image is shipped into the `linux-bootimage-VERSION-VENDOR-DEVICE`
package.

You can pick up the boot.img by extracting the package with `dpkg-deb` or
by picking up directly from the compiled artifacts (`out/KERNEL_OBJ/boot.img`).

The kernel image already embeds the generic Halium initramfs.

Committing changes
------------------

When you're happy with the kernel, be sure to commit the following files in git:

* debian/source/
* debian/control
* debian/rules
* debian/compat
* debian/kernel-info.mk

...and then push your `bookworm` branch for others to enjoy.
