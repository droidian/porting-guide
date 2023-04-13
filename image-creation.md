Image creation
==============

Devices released before the treble/GSI move require a system image to be built because the default image will not provide the required libraries to the system.

Table of contents
-----------------

* [Summary](#summary)
* [Prerequisites](#prerequisites)
* [Build environment](#build-environment)
* [Creating a manifest file](#creating-a-manifest-file)
* [Initializing the build environment](#initializing-the-build-environment)
* [Device tree modifications](#device-tree-modifications)
* [Compiling](#compiling)
* [Obtaining the image](#obtaining-the-image)
* [Testing out the system image](#testing-out-the-system-image)
* [Packaging up the system image](#packaging-up-the-system-image)

Summary
-------

Halium based operating systems use Android drivers to make use of the systems hardware. doing so can make the process of porting a lot faster and smoother.

To do so it uses an Android container running a very trimmed down version of Android in the background. Then uses middleware and various patches to system software to make use of those loaded drivers.

on GSI systems a system image which makes use of the vendor partition for system firmware can be used. but on older devices without a vendor partition (non-Treble), a system image has to be built from scratch.

Currently this method only works on devices with Android 9 (halium-9.0). As an example a device released with Android 7 but with Android 9 ported to the device.

Prerequisites
-------------

* Device kernel source
* Device tree source
* Device vendor tree source (if applicable)

Build environment
-----------------

Before we start, It should be noted that the Android build system will be well over 50GB when building and is about 35GB on start without any modifications. So make sure you have enough storage before starting the process.

First create a directory for the Halium/Android build system

	(host)$ mkdir droidian && cd droidian

and initialize the correct manifest file for your Halium version (in this case it will he halium-9.0)

	(host)$ repo init -u https://github.com/Halium/android -b halium-9.0 --depth=1

Now you are ready to clone into all the repositories and download the source code for the required repositories. (This will take some time)

	(host)$ repo sync -c

If you have a fast connection you can increase the number of workers with -j

	(host)$ repo sync -c -j 16

Creating a manifest file
------------------------

After downloading all the sources you'll need to write a manifest file for your device.
A manifest file is a file in which the location to all your sources on the internet and the location of all your sources on your local disk is stored.

Manifest files are stored in halium/devices/manifests and the file name has the format `vendor_device.xml` in which device is the codename of your device
As an example for Xiaomi Redmi Note 7 it will be `xiaomi_lavender.xml`

Your manifest file should have this as the starter code and everything should be appended

```
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
    
</manifest>
```

### Remotes

Remotes can be created to mark which git instance each project should be cloned from. as an example

`<remote name="github" fetch="https://github.com/" revision="halium-9.0" />`

This means that each time github is used as the remote repository will be searched for in github.com and the branch will always be halium-9.0 (revision can be removed from remote and can be moved to project entries)

The only condition for a remote is the fetch value must be a git instance.

## Project

Project entries indicate the repository to clone into. as an example

`<project path="device/lge/hammerhead" name="droidian-hammerhead/android_device_lge_hammerhead" remote="github" />`

This line indicates that the build script should look at the remote github (which is https://github.com/) and clone into the repository `droidian-hammerhead/android_device_lge_hammerhead` (https://github.com/droidian-hammerhead/android_device_lge_hammerhead) and store it at device/lge/hammerhead from the root of the build system.

Because we have revision="halium-9.0" it will also make sure to clone into the halium-9.0 branch.
revision can also be added to the project line instead of the remote if not all repositories have the same branch name.

`<project path="device/lge/hammerhead" name="droidian-hammerhead/android_device_lge_hammerhead" remote="github" revision="halium-9.0" />`

After adding all the entries for your device (kernel, device, vendor and possibly other entries such as slsi for Samsung devices), you can move on to the next step.

### Remove project

remove-project entires are used to remove a directory or file in the build system. as an example

`<remove-project name="hybris-patches" />`

This line removes hybris-patches directory. It can then be replaced by another project

`<project path="hybris-patches" name="Halium/hybris-patches" revision="halium-9.0-arm32" />`

On this example it is replaced by an arm32 version of hybris patches needed for devices with a 32 bit CPU as the default hybris-patches is arm64.

Initializing the build environment
----------------------------------

After completing your manifest file you can use the setup script to set your device as the target for building.

Make sure to run these commands at the root of your droidian directory

	(host)$ halium/devices/setup CODENAME

replace CODENAME with your codename

Then apply the hybris patches for the build system

	(host)$ hybris-patches/apply-patches.sh --mb

Now all we have to do to have a ready to build environment is to set the set up the environment variables

	(host)$ source build/envsetup.sh
	(host)$ breakfast CODENAME

replace CODENAME with your codename

Device tree modifications
-------------------------

For Halium 9, we need the system image to be built as system-as-root 

`BOARD_BUILD_SYSTEM_ROOT_IMAGE := true`

As this change makes the android root read-only, we may have to ship mount points for some important partitions like firmware, persist with the system image, for this we can use the line

```
BOARD_ROOT_EXTRA_FOLDERS := \
 /firmware \
 /dsp \
 /persist
```

You may need to add more mount points depending on your device. After successful boot do `ls -la /` and add folders corresponding to broken symlinks.

You may also need to comment out java (jar) libraries and executables from building as we don't need those in our environment.

Compiling
---------

To build the system image

	(host)$ mka e2fsdroid
	(host)$ mka systemimage

Obtaining the image
-------------------

If everything went smoothly without any errors, you should be able to find all the images in `out/target/product/CODENAME` where CODENAME is your codename.
You should see `system.img`.

We won't be using the `halium-boot.img` created here but the kernel compilation guide should be used for building a proper boot image.

Testing out the system image
----------------------------

To test this image in Droidian, it should first be converted from a sparse image to a raw image

	(host)$ simg2img system.img system-raw.img

Now it can safely be moved to our Droidian installation.

Assuming device is already booted into Droidian, scp can be used to move the image.

	(host)$ scp system-raw.img droidian@10.15.19.82:~/

Now ssh into your device

	(host)$ ssh droidian@10.15.19.82

And move the image to `/var/lib/lxc/android/`

	(device)$ sudo cp system-raw.img /var/lib/lxc/android/android-rootfs.img

If device is not booted up yet, rootfs can be modified from recovery by mounting it (`/data/rootfs.img`) and copying the image

	(host)$ adb push system-raw.img /data/

Now from recovery shell

	(recovery)$ mkdir /tmp/mpoint
	(recovery)$ mount /data/rootfs.img /tmp/mpoint
	(recovery)$ cp /data/system-raw.img /tmp/mpoint/var/lib/lxc/android/android-rootfs.img

After the system image is tested and confirmed working then you can move on to [Creating a rootfs](./rootfs-creation.md)

Packaging up the system image
-----------------------------

You'll have to package your system image to use later on when building your rootfs in [Rootfs creation](./rootfs-creation)

The main note here is that when building your debian package with your system image, it should have `android-system-gsi-28` as a conflicting package.

You should also make sure to move the the system image to `/var/lib/lxc/android/android-rootfs.img` upon installation.
