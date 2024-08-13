内核编译
==================

遗憾的是，原版 Android 内核不足以运行 Droidian。

好消息是，对于支持 GSI 的设备，通常只需进行内核更改。

目录
-----------------

* [总结](#summary)
* [先决条件](#prerequisites)
* [软件包配置](#package-bring-up)
* [内核信息选项](#kernel-info-options)
* [内核适配](#kernel-adaptation)
* [编译](#compiling)
* [获取启动镜像](#obtaining-the-boot-image)
* [提交更改](#committing-changes)

总结
-------

Droidian 运行在 [halium](https://halium.org) 上。
如果你的设备是 Android 9 或更高版本发布的，那么就有可能将 Droidian 移植到该设备上。
如果它已经有 halium-9.0 及以上版本的兼容内核，Droidian 很可能会在没有太多修改的情况下正常工作。

如果你的设备是 Android 8.1 或 9，可能支持 GSI（通用系统镜像），这样可以使用已经提供的、带有 Halium 补丁的通用 Android 系统镜像。
这将显著减少移植者需要做的工作量。

如果你的设备不支持 GSI，你还需要编译一个打了补丁的系统镜像。
有关镜像创建的文档可以在 [此指南](./image-creation.md) 中找到。

在 Halium 上，Android 内核是通过标准的 Android 工具链构建的。
虽然这样做是合理的，但对于支持 GSI 的设备，这可能是浪费时间，因为通常只需要更改内核。

因此，Droidian 使用不同的方法来编译内核——这样做的好处是你可以免费获得打包功能，从而可以通过 APT 进行 OTA 内核升级（如果你愿意的话）。

请注意，本指南假设你将使用 Android 提供的预编译工具链在 x86_64（amd64）机器上交叉编译 arm64 Android 内核。
禁用交叉编译并使用标准 Debian 工具链进行编译是非常简单的。

使用这种方法，你还可以编译和打包主线内核。

使用本指南打包的一个示例内核是 [F(x)tec Pro1 的 Android 内核](https://github.com/droidian-devices/linux-android-fxtec-pro1/tree/droidian/debian)。

先决条件
-------------

* 设备内核源代码
* 一个兼容 Halium 的内核 defconfig
* Docker

如果你还没有兼容 Halium 的内核，你应该根据指南后面 [内核适配](#kernel-adaptation) 的建议修改内核的配置。在此之前 **确保你的内核源代码可以编译并能启动 Android**。

软件包配置
----------------

假设你的内核源代码位于 `~/droidian/kernel/vendor/device`，你应该创建一个新分支来存放 Debian 打包文件。

我们还假设你希望将生成的包放在 `~/droidian/packages`。

Droidian 工具需要内核源代码有一个有效的 git 目录结构（即一个从 git 仓库克隆的内核）。

	(host)$ KERNEL_DIR="$HOME/droidian/kernel/vendor/device"
	(host)$ PACKAGES_DIR="$HOME/droidian/packages"
	(host)$ mkdir -p $PACKAGES_DIR
	(host)$ cd $KERNEL_DIR
	(host)$ git checkout -b droidian

现在是启动 Docker 容器的时候了。

	(host)$ docker run --rm -v $PACKAGES_DIR:/buildd -v $KERNEL_DIR:/buildd/sources -it quay.io/droidian/build-essential:current-amd64 bash

在 Docker 容器中，安装 `linux-packaging-snippets`，它提供了示例 `kernel-info.mk` 文件。

	(docker)# apt-get install linux-packaging-snippets

并创建骨架打包文件：

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

现在编辑 `debian/kernel-info.mk` 以匹配你的内核设置。

大多数默认设置足以使用 Pie 工具链构建 Android 内核，因此你可能需要更改特定于设备的设置（例如供应商、名称、cmdline、defconfig 和各种偏移量）。

通过解压一个已经构建的 `boot.img`，可以找到所有偏移量。

内核信息选项
-------------------

unpackbootimg 的语法如下

	(docker)# unpackbootimg --boot_img boot.img

或者使用 AOSP 版本的 unpackbootimg

	(docker)# unpackbootimg -i boot.img

### kernel-info.mk 条目

* `KERNEL_BASE_VERSION` 是可以在内核源代码根目录的 Makefile 中查看的内核版本。

例如

```
VERSION = 4
PATCHLEVEL = 14
SUBLEVEL = 221
```

将会是 4.14.221

* `KERNEL_DEFCONFIG` 是在 arch/YOURARCH/configs 中找到的 defconfig 文件名

* `KERNEL_IMAGE_WITH_DTB` 决定是否在内核中包含 dtb 文件。如果此选项被设置，则还需要设置 `KERNEL_IMAGE_DTB`。如果没有，将尝试找到它。

* `KERNEL_IMAGE_DTB` 是 dtb 文件的路径，可以在 arch/YOURARCH/boot/dts/SOC/ 中找到

* `KERNEL_IMAGE_WITH_DTB_OVERLAY` 决定是否构建 dtbo 文件。如果此选项被设置，则还需要设置 `KERNEL_IMAGE_DTB_OVERLAY`。如果没有，将尝试找到它。

* `KERNEL_IMAGE_DTB_OVERLAY` 是 dtbo 文件的路径，可以在 arch/YOURARCH/boot/dts/SOC/ 中找到

* `KERNEL_PREBUILT_DT` 适用于有预构建 DT 镜像的设备（如三星），并且需要在内核树中指定路径。

所有这些值可以通过提取 boot 镜像并使用 unpackbootimg 查看

* `KERNEL_BOOTIMAGE_CMDLINE` 对应于“命令行参数”或“BOARD_KERNEL_CMDLINE”。`console=tty0` 和 `droidian.lvm.prefer` 应该被追加到 cmdline 中。确保从 cmdline 中删除任何 `systempart` 条目。

* `KERNEL_BOOTIMAGE_PAGE_SIZE` 对应于“页面大小”或“BOARD_PAGE_SIZE”

* `KERNEL_BOOTIMAGE_BASE_OFFSET` 对应于“基址”或“BOARD_KERNEL_BASE”

* `KERNEL_BOOTIMAGE_KERNEL_OFFSET` 对应于“内核加载地址”或“BOARD_KERNEL_OFFSET”

* `KERNEL_BOOTIMAGE_INITRAMFS_OFFSET` 对应于“ramdisk 加载地址”或“BOARD_RAMDISK_OFFSET”

* `KERNEL_BOOTIMAGE_SECONDIMAGE_OFFSET` 对应于“第二启动加载地址”或“BOARD_SECOND_OFFSET”

* `KERNEL_BOOTIMAGE_TAGS_OFFSET` 对应于“内核标签加载地址”或“BOARD_TAGS_OFFSET”

* `KERNEL_BOOTIMAGE_DTB_OFFSET` 对应于“dtb 地址”或“BOARD_DTB_OFFSET”

虽然此选项仅在内核头文件版本 2 中是必需的，否则可以被注释掉。

* `KERNEL_BOOTIMAGE_VERSION` 关联到内核头文件版本。设备发布时 Android 8 及以下为 0，Android 9 为 1，Android 10 为 2，Android 11 为 2，而 GKI 设备为 3。

* 对于三星设备，`DEVICE_VBMETA_IS_SAMSUNG` 必须设置为 1。

* 对于大多数 Android 9 及以上发布的设备，`BUILD_CC` 是 clang，但如果你的内核在使用 `clang` 时无法编译，你可以尝试将值更改为 `aarch64-linux-android-gcc-4.9` 以使用 gcc 编译。

* 如果你的设备需要的工具链不包含在构建系统中，你可以手动下载工具链并将路径添加到 `BUILD_PATH`。

* `DEB_BUILD_FOR` 和 `KERNEL_ARCH` 应根据设备架构进行更改。

### 启用自动启动分区闪存

安装构建的包意味着内核及其模块已经到位——但用户仍然需要将其闪存到启动分区。

如果你希望启用自动闪存（通过 [flash-bootimage](https://github.com/droidian/flash-bootimage)），可以通过将 `FLASH_ENABLED` 设置为 1（这是默认值）来实现。

如果你的设备不支持 A/B 更新，请确保将 `FLASH_IS_LEGACY_DEVICE` 设置为 1。

注意，你需要指定一些设备信息，以便 `flash
