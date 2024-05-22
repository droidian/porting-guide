Rootfs creation
===============

To make the installation easier and more consistent we build device specific rootfs images.

Table of contents
-----------------

* [Summary](#summary)
* [Prerequisites](#prerequisites)
* [Package creation](#package-creation)
* [Building the rootfs](#building-the-rootfs)
* [Automating nightly images](#automating-nightly-images)

Summary
-------

After debugging you should start building a rootfs specific to your device to make installations more consistent and user friendly.

This process involves creating an adaptation package and an apt repository

Adaptation package is a debian package that holds all the device specific configurations and custom binaries and scripts needed for the device to function correctly.

To be able to do OTA updates, you have to set up an apt repository either on some platform like GitHub or a server.

Creation of the apt repository is out of the scope of this tutorial.

Prerequisites
-------------

* Device specific files
* Docker

Dependencies
------------

dpkg-dev gpg git and apt-utils are required to be installed on your host

	(host)# apt update && apt install dpkg-dev gpg apt-utils

Package creation
----------------

First clone into the droidian build tools repository

	(host)$ git clone https://github.com/droidian-releng/droidian-build-tools/ && cd droidian-build-tools/bin

Now create a template for your device. of course replace vendor, codename, ARCH and APIVER

APIVER can be 28, 29 or 30 depending on which Android version you're using as your base

ARCH can be either arm64, armhf or amd64

	(host)$ ./droidian-new-device -v vendor -n codename -c ARCH -a APIVER -r phone -d droidian

As you will be building for a device with another architecture, you must initialize qemu-user-static to enable emulation for docker

	(host)$ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

Go to the newly created adaptation directory

	(host)$ cd droidian/vendor/codename/packages/adaptation-vendor-codename/

Now put all your device specific files under `sparse/` with a linux directory structure

Make sure to put your repository's URL in `~/droidian-build-tools/bin/droidian/vendor/codename/packages/adaptation-vendor-codename/sparse/usr/lib/adaptation-vendor-model/sources.list.d/community-vendor-device.list` with this format

`deb [signed-by=/usr/share/keyrings/codename.gpg] https://YOURREPO.TLD bookworm main`

Then copy your gpg file to `~/droidian-build-tools/bin/droidian/vendor/codename/packages/adaptation-vendor-codename/sparse/usr/share/keyrings/codename.gpg`. Make sure to replace codename.

After copying all your files build the deb package

	(host)$ ~/droidian-build-tools/bin/droidian-build-package

Make sure to copy all the files from `~/droidian-build-tools/bin/droidian/vendor/codename/droidian/apt` to the repository you specified in `~/droidian-build-tools/bin/droidian/vendor/codename/packages/adaptation-vendor-codename/sparse/usr/lib/adaptation-vendor-model/sources.list.d/community-vendor-device.list`

Now at this point you can use `package-sideload-create` on your droidian device to create a recovery flashable package from those two deb packages.

After copying the deb files to your personal repository, add that repository to your device by adding the sources list file to your device then do

	(device)# apt update
	(device)# package-sideload-create adaptation-droidian-codename.zip adaptation-vendor-codename adaptation-vendor-codename-configs

At the end you should get a zip file which is flashable from recovery and has all your changes.

Also make sure to commit your repository/changes to a git repository.

Building the rootfs
-------------------

Before building the rootfs make sure to add your `linux-*.deb` packages you have build during the kernel compilation process to `~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/droidian/community_devices.yml` as a package entry.

As an alternative solution you can try adding those packages as a dependency to your adaptation package in `~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/debian/control`.

First pull the rootfs-builder docker image

	(host)$ docker pull quay.io/droidian/rootfs-builder:current-amd64

Now run debos in the docker container rootfs-builder to build the rootfs. Make sure to replace the placeholder values

	(host)$ cd ~/droidian-build-tools/droidian/vendor/codename/droidian
	(host)$ mkdir images
	(host)$ docker run --privileged -v $PWD/images:/buildd/out -v /dev:/host-dev -v /sys/fs/cgroup:/sys/fs/cgroup -v $PWD:/buildd/sources --security-opt seccomp:unconfined quay.io/droidian/rootfs-builder:current-amd64 /bin/sh -c 'cd /buildd/sources; DROIDIAN_VERSION="nightly" ./generate_device_recipe.py vendor_codename ARCH phosh phone APIVER && debos --disable-fakemachine generated/droidian.yaml'

If everything builds fine you should have your LVM fastboot flashable rootfs image in `~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/droidian/images/`.

Automating nightly images
-------------------------

You can automate your builds with GitHub actions to generate new images every day for your device with your changes. Take [this actions yml file](https://github.com/droidian-onclite/droidian-images/blob/bookworm/.github/workflows/release.yml) as an example.

You can replicate nightly builds by replacing the values in `community_devices.yml` and files in `apt/`.
