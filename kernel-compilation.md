内核编译
=================

遗憾的是，现有的 Android 内核不足以运行 Droidian。

好消息是，在支持 GSI 的设备上，通常只需要更改内核。

目录
-----------------

* [摘要](#摘要)
* [先决条件](#先决条件)
* [包启动](#包启动)
* [内核信息选项](#内核信息选项)
* [内核适配](#内核适配)
* [编译](#编译)
* [获取启动镜像](#获取启动镜像)
* [提交更改](#提交更改)

摘要
--------

Droidian 在 [halium](https://halium.org) 上运行。
如果您的设备运行的是 Android 9 或更高版本，则可以将 Droidian 移植到该设备​​。
如果它已经具有 halium-9.0 及以上版本的兼容 halium 的内核，则 Droidian 无需太多修改即可运行。

如果您的设备预装了 Android 8.1 或 9，则它可能具有 GSI（通用系统映像）功能，因此可以使用已应用的 Halium 补丁的现有通用 Android 系统映像。
这将大大减少搬运工必须做的工作量。

如果您的设备不支持 GSI，您还需要编译修补的系统映像。
图像创建的文档可以在[本指南](./image-creation.md)中找到

在 Halium 上，Android 内核是通过标准 Android 工具链构建的。
虽然这是有道理的，但在支持 GSI 的设备上这可能会浪费时间，因为通常只需要更改内核。

因此，Droidian 使用不同的方法来编译内核 - 好处是您可以免费打包，以便可以通过 APT 无线升级内核（如果您愿意）。

请注意，本指南假设您要交叉编译 arm64
使用 Android 提供的 x86_64 (amd64) 计算机上的 Android 内核
Droidian 存储库中提供了预编译的工具链。
禁用交叉编译和使用标准编译很简单
Debian 工具链。

使用此方法您还可以编译和打包主线内核。

使用本指南打包的示例内核是 [F(x)tec Pro1 的 Android 内核](https://github.com/droidian-devices/linux-android-fxtec-pro1/tree/droidian/debian)。

先决条件
-------------

* 设备内核源码
* 符合 Halium 标准的内核 defconfig
* Docker

如果您还没有兼容 Halium 的内核，您应该按照指南后面的[内核适配](#内核适配) 中的建议修改内核的配置。在执行此操作之前**确保您的内核源代码可以构建并且可以启动 Android**。

包启动
----------------

假设您的内核源位于“~/droidian/kernel/vendor/device”中，
您应该创建一个新分支来容纳 Debian 软件包。

我们还假设您希望将生成的包放在“~/droidian/packages”中。

Droidian 工具期望内核源代码具有工作的 git 目录结构（是从 git 存储库克隆的内核）。

	(host)$ KERNEL_DIR="$HOME/droidian/kernel/vendor/device"
	(host)$ PACKAGES_DIR="$HOME/droidian/packages"
	(host)$ mkdir -p $PACKAGES_DIR
	(host)$ cd $KERNEL_DIR
	(host)$ git checkout -b droidian

现在是时候启动 Docker 容器了。

	(host)$ docker run --rm -v $PACKAGES_DIR:/buildd -v $KERNEL_DIR:/buildd/sources -it quay.io/droidian/build-essential:current-amd64 bash

在 Docker 容器内，安装 `linux-packaging-snippets`，
提供示例“kernel-info.mk”文件。

	(docker)# apt-get install linux-packaging-snippets

并创建框架package：

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

现在编辑 `debian/kernel-info.mk` 以匹配您的内核设置。

大多数默认值足以使用 Pie 构建 Android 内核
工具链，因此您可能需要更改特定于设备的设置（例如
如 vendor、name、cmdline、defconfig 和各种偏移量）。

通过使用 unpackbootimg 解压已经构建的“boot.img”，可以找到所有偏移量。

内核信息选项
-------------------

unpackbootimg 语法如下

	(docker)# unpackbootimg --boot_img boot.img

或者 unpackbootimg 的 AOSP 版本

	(docker)# unpackbootimg -i boot.img

### kernel-info.mk 条目

* `KERNEL_BASE_VERSION` 是内核版本，可以在内核源代码根目录下的 Makefile 中查看。

举个例子

```
VERSION = 4
PATCHLEVEL = 14
SUBLEVEL = 221
```

将于 4.14.221

* `KERNEL_DEFCONFIG` 是在 arch/YOURARCH/configs 中找到的 defconfig 文件名

* `KERNEL_IMAGE_WITH_DTB` 决定是否在内核中包含 dtb 文件。如果设置此选项，还需要设置“KERNEL_IMAGE_DTB”。如果没有，请尝试找到它

* `KERNEL_IMAGE_DTB` 是 dtb 文件的路径，可以在 arch/YOURARCH/boot/dts/SOC/ 中找到

* `KERNEL_IMAGE_WITH_DTB_OVERLAY` 决定是否构建 dtbo 文件。如果设置此选项，还需要设置“KERNEL_IMAGE_DTB_OVERLAY”。如果没有请-=尝试找到它

* `KERNEL_IMAGE_DTB_OVERLAY` 是 dtbo 文件的路径，可以在 arch/YOURARCH/boot/dts/SOC/ 中找到

* `KERNEL_PREBUILT_DT` 适用于具有预构建 DT 映像的设备（例如三星），并采用内核树中的路径。

所有这些值都可以通过使用 unpackbootimg 提取启动映像来查看

* `KERNEL_BOOTIMAGE_CMDLINE` 对应于“cmdline”或“BOARD_KERNEL_CMDLINE” `console=tty0` 和 `droidian.lvm.prefer` 应附加到命令行。确保从命令行中删除任何“systempart”条目。

* `KERNEL_BOOTIMAGE_PAGE_SIZE` 对应 "page size" 或者 "BOARD_PAGE_SIZE"

* `KERNEL_BOOTIMAGE_BASE_OFFSET` 对应 "base" 或者 "BOARD_KERNEL_BASE"

* `KERNEL_BOOTIMAGE_KERNEL_OFFSET` 对应 "kernel load address" 或者 "BOARD_KERNEL_OFFSET"

* `KERNEL_BOOTIMAGE_INITRAMFS_OFFSET` 对应 "ramdisk load address" 或者 "BOARD_RAMDISK_OFFSET"

* `KERNEL_BOOTIMAGE_SECONDIMAGE_OFFSET` 对应 "second bootloader load address" 或者 "BOARD_SECOND_OFFSET"

* `KERNEL_BOOTIMAGE_TAGS_OFFSET` 对应 "kernel tags load address" 或者 "BOARD_TAGS_OFFSET"

* `KERNEL_BOOTIMAGE_DTB_OFFSET` 对应       "dtb address" 或者 "BOARD_DTB_OFFSET"

虽然此选项仅适用于内核标头版本2。 它可以被注释，否则。

*'KERNEL_BOOTIMAGE_VERSION'与内核头版本相关。 使用Android8及更低版本推出的设备为0，Android9为1，Android10为2，Android11为2，GKI设备为3。

*对于三星设备的DEVICE_VBMETA_IS_SAMSUNG'必须设置为1。

*'BUILD_CC'对于大多数使用Android9及以上版本启动的设备来说是clang，但如果您的内核无法使用`clang`构建，您可以尝试将值更改为`aarch64-linux-android-gcc-4.9`以使用gcc构建。

*如果您的设备需要构建系统中未包含的工具链，则可以手动下载工具链并将路径添加到"BUILD_PATH"。

*'DEB_BUILD_FOR'和'KERNEL_ARCH'应根据设备架构进行更改。

###Enabling automatic boot partition flashing

安装构建的软件包意味着内核及其模块是
到位-但用户仍然应该将其闪存到他们的启动分区。

如果您希望启用自动闪烁（通过[flash-bootimage]（https://github.com/droidian/flash-bootimage)),
您可以通过将`FLASH_ENABLED`设置为1（这是默认值）来执行此操作。

如果您的设备不支持A/B更新，请务必将`FLASH_IS_LEGACY_DEVICE'设置为1。

请注意，您需要指定一些设备信息，以便在设备上运行时"flash-bootimage"可以交叉检查:

* `FLASH_INFO_MANUFACTURER`：`ro.product.vendor.manufacturer`的值
安卓属性。在正在运行的 Droidian 系统上，您可以通过以下方式获取它

	(device)$ sudo android_getprop ro.product.vendor.manufacturer

* `FLASH_INFO_MODEL`: `ro.product.vendor.model` 的值
安卓属性。在正在运行的 Droidian 系统上，您可以通过以下方式获取它

	(device)$ sudo android_getprop ro.product.vendor.model

* `FLASH_INFO_CPU`: 来自的相关信息 `/proc/cpuinfo`.

如果`FLASH_INFO_MANUFACTURER`或`FLASH_INFO_MODEL'未定义（它们都是
需要检查Android属性），'flash-bootimage`
将检查'FLASH_INFO_CPU'。

如果未指定设备特定信息，内核升级将会失败。

F(x)tec Pro1 的示例：

```
FLASH_ENABLED = 1
FLASH_INFO_MANUFACTURER = Fxtec
FLASH_INFO_MODEL = QX1000
FLASH_INFO_CPU = Qualcomm Technologies, Inc MSM8998
```

### 工具链

Droidian ships

* gcc 4.9 (legacy kernels)
* clang-android-6.0-4691093 (recommended toolchain for android9)
* clang-android-9.0-r353983c (recommended toolchain for android10)
* clang-android-10.0-r370808 (recommended toolchain for android11)
* clang-android-12.0-r416183b (recommended toolchain for android12-12.1)
* clang-android-14.0-r450784d (recommended toolchain for android13)

要使用 `clang-android-6.0-4691093` 将其添加到 `DEB_TOOLCHAIN` 并将 `BUILD_PATH` 设置为以下值

`/usr/lib/llvm-android-6.0-4691093/bin`

要使用 `clang-android-9.0-r353983c` 将其添加到 `DEB_TOOLCHAIN` 并将 `BUILD_PATH` 设置为以下值

`/usr/lib/llvm-android-9.0-r353983c/bin`

要使用 `clang-android-10.0-r370808` 将其添加到 `DEB_TOOLCHAIN` 并将 `BUILD_PATH` 设置为以下值

`/usr/lib/llvm-android-10.0-r370808/bin`

要使用 `clang-android-12.0-r416183b` 将其添加到 `DEB_TOOLCHAIN` 并将 `BUILD_PATH` 设置为以下值

`/usr/lib/llvm-android-12.0-r416183b/bin`

要使用 `clang-android-14.0-r450784d` 将其添加到 `DEB_TOOLCHAIN` 并将 `BUILD_PATH` 设置为以下值

`/usr/lib/llvm-android-14.0-r450784d/bin`

如果您使用的是较旧的设备并且您的内核未使用任何 clang 工具链进行编译，您可以回退到 GCC

要使用 GCC，请将“BUILD_CC”从“clang”更改为“aarch64-linux-android-gcc-4.9”

请记住，任何高于 4.4 的内核都应该可以使用 clang 正常编译，但总有例外。

### Build target

构建目标被传递给"make"，它告诉make命令我们在构建结束时的期望。

默认构建目标是"图像"。生成图像并在制作gzip存档后添加dtb图像的gz`。

'形象。gz-dtb'也可用于需要将DTB附加到内核末尾的旧设备。

如果您的设备需要预先构建的dt，也可以使用"图像"

您可以通过查看设备的设备树来查找设备所需的构建目标。

### initramfs hooks

如果 initramfs 出现任何问题（例如 unl0kr 在启用加密时显示黑屏），可以将脚本添加到将包含在 ramdisk 中的包中。

[这个打包](https://github.com/droidian-devices/linux-android-fxtec-pro1x/blob/droidian/debian/initramfs-overlay/scripts/halium-hooks)可以作为参考。

以下添加了一个将在启动时在 initramfs 中运行的函数。

任何有助于系统启动的命令都可以添加到脚本中。

### build rules

在内核编译阶段（打包期间）之后失败的构建中，可以忽略错误。

构建序列可以用`override_dh_sequence'来改变（确保用自己的序列替换序列）

作为例子

```
override_dh_dwz:
override_dh_strip:
override_dh_makeshlibs:
```

可以添加到 `debian/rules` 的末尾。

建议稍后返回并修复构建错误，而不是忽略它们，因为它们可能会在将来引起各种问题。

内核适配
-----------------
Droidian 提供了配置片段。

`'KERNEL_CONFIG_USE_FRAGMENTS=1'可以在`kernel-info.mk'在构建时将defconfig片段包含在defconfig中。

defconfig片段应该放在内核源的根目录中，位于一个名为droidian的目录中。 [pro1x](https://github.com/droidian-devices/linux-android-fxtec-pro1x/tree/droidian/droidian)可以作为参考。

可以启用'KERNEL_CONFIG_USE_DIFFCONFIG`以使用python脚本`diffconfig'来比较片段和主defconfig。

要使用diffconfig，应该像这样提供diff文件

`KERNEL_PRODUCT_DIFFCONFIG = diff_file`

如果LXC无法启动，您可以在设备首次启动后使用`lxc-checkconfig`来检查可能需要的其他选项。

编译
---------

现在`kernel-info.mk`已经被修改了，唯一剩下的就是
就是真正编译内核。

首先，（重新）创建 `debian/control` 文件：

	(docker)# rm -f debian/control
	(docker)# debian/rules debian/control

现在一切就绪，您可以使用 `releng-build-package` 开始构建：

	(docker)# RELENG_HOST_ARCH="arm64" releng-build-package

交叉构建时需要'RELENG_HOST_ARCH'变量。

如果一切顺利，您将在`$PACKAGES_DIR`中找到生成的包。

DTB/DTBO 编译失败
----------------------------

如果 DTB/DTBO 编译失败，最好尝试使用外部 dtc 编译器而不是内核树中存在的编译器。

要在构建控制文件后执行此操作，请将“device-tree-compiler”添加到“Build-Depends”，然后在“debian/rules”中的 include 行添加之后

`BUILD_COMMAND := $(BUILD_COMMAND) DTC_EXT=/usr/bin/dtc`

获取启动镜像
------------------------

启动映像被运送到“linux-bootimage-VERSION-VENDOR-DEVICE”包中。

您可以通过使用“dpkg-deb”解压软件包来获取 boot.img 或
通过直接从编译的工件（`out/KERNEL_OBJ/boot.img`）中获取。

内核镜像已经嵌入了 Droidian initramfs。

确保保存所有“linux-*.deb”软件包，因为您将在指南中进一步需要这些软件包。

提交更改
------------------

当您对内核感到满意时，请务必将您的更改以及 debian 打包提交到 git 存储库：

* debian/source/
* debian/control
* debian/rules
* debian/compat
* debian/kernel-info.mk

...然后推送你的“droidian”分支供其他人享用。
