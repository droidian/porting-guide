根文件系统创建
===============

为了使安装更容易、更一致，我们构建了设备特定的 rootfs 映像。

目录
-----------------

* [摘要](#summary)
* [先决条件](#先决条件)
* [包创建](#package-creation)
* [构建 rootfs](#building-the-rootfs)
* [夜间自动构建](#automating-nightly-images)

概括
--------

调试后，您应该开始构建特定于您的设备的 rootfs，以使安装更加一致和用户友好。

此过程涉及创建适配包和 apt 存储库

适配包是一个 debian 包，其中包含设备正常运行所需的所有设备特定配置以及自定义二进制文件和脚本。

为了能够进行 OTA 更新，您必须在 GitHub 等平台或服务器上设置 apt 存储库。

apt 存储库的创建超出了本教程的范围。

先决条件
-------------

* 设备特定文件 (Device Specific files)
* Docker

依赖关系
------------

主机上需要安装 dpkg-dev gpg git 和 apt-utils

	(host)# apt update && apt install dpkg-dev gpg apt-utils

包创建
----------------

首先克隆 droidian 构建工具存储库

	(host)$ git clone https://github.com/droidian-releng/droidian-build-tools/ && cd droidian-build-tools/bin

现在为您的设备创建一个模板。当然替换 vendor、codename、ARCH 和 APIVER

APIVER 可以是 28、29 或 30，具体取决于您用作基础的 Android 版本

ARCH 可以是 arm64、armhf 或 amd64

	(host)$ ./droidian-new-device -v vendor -n codename -c ARCH -a APIVER -r phone -d droidian

由于您将构建具有其他架构的设备，因此必须初始化 qemu-user-static 以启用 docker 模拟

	(host)$ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

进入新创建的适配目录

	(host)$ cd droidian/vendor/codename/packages/adaptation-vendor-codename/

现在将所有设备特定文件放在具有 Linux 目录结构的“sparse/”下

确保将存储库的 URL 放入 `~/droidian-build-tools/bin/droidian/vendor/codename/packages/adaptation-vendor-codename/sparse/usr/lib/adaptation-vendor-model/sources.list.d /community-vendor-device.list` 采用这种格式

`deb [signed-by=/usr/share/keyrings/codename.gpg] https://YOURREPO.TLD bookworm`

然后将 gpg 文件复制到 `~/droidian-build-tools/bin/droidian/vendor/codename/packages/adaptation-vendor-codename/sparse/usr/share/keyrings/codename.gpg`。确保替换代号。

复制所有文件后构建 deb 包

	(host)$ ~/droidian-build-tools/bin/droidian-build-package

确保将`~/droidian-build-tools/bin/droidian/vendor/codename/droidian/apt`中的所有文件复制到您在`~/droidian-build-tools/bin/droidian/vendor/中指定的存储库代码名/packages/adaptation-vendor-codename/sparse/usr/lib/adaptation-vendor-model/sources.list.d/community-vendor-device.list`

现在，您可以在 droidian 设备上使用 `package-sideload-create` 从这两个 deb 包创建恢复可闪存包。

将 deb 文件复制到您的个人存储库后，通过将源列表文件添加到您的设备，将该存储库添加到您的设备，然后执行以下操作
> 请参阅我们关于托管包存储库的[指南](./host-package-repo.md)以获取详细说明。

	(device)# apt 更新
	(device)# package-sideload-create adjustment-droidian-codename.zip adjustment-vendor-codename adjustment-vendor-codename-configs

最后，您应该获得一个 zip 文件，该文件可通过恢复进行刷新并包含您的所有更改。

还要确保将您的存储库/更改提交到 git 存储库。

构建 rootfs
-------------------

在构建 rootfs 之前，请确保将您在内核编译过程中构建的 `linux-*.deb` 软件包添加到 `~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/droidian /community_devices.yml` 作为包条目。

作为替代解决方案，您可以尝试将这些包作为依赖项添加到“~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/debian/control”中的适配包。

首先拉取 rootfs-builder docker 镜像

	(host)$ docker pull quay.io/droidian/rootfs-builder:current-amd64

现在在 docker 容器 rootfs-builder 中运行 debos 来构建 rootfs。确保替换占位符值

	(host)$ cd ~/droidian-build-tools/droidian/vendor/代号/droidian
	(host)$ mkdir images
	(host)$ docker run --privileged -v $PWD/images:/buildd/out -v /dev:/host-dev -v /sys/fs/cgroup:/sys/fs/cgroup -v $PWD:/ buildd/sources --security-opt seccomp:unconfined quay.io/droidian/rootfs-builder:current-amd64 /bin/sh -c 'cd /buildd/sources; DROIDIAN_VERSION="nightly" ./generate_device_recipe.pyvendor_codename ARCH phosh phone APIVER && debos --disable-fakemachine generated/droidian.yaml'

如果一切正常，您应该在“~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/droidian/images/”中拥有 LVM fastboot 可 flash  rootfs 映像。

夜间自动构建
------------------------

您可以使用 GitHub Actions 自动化构建，每天根据您的更改为您的设备生成新图像。以[此yml文件](https://github.com/droidian-onclite/droidian-images/blob/bookworm/.github/workflows/release.yml) 为例。

设置夜间构建：

**分支存储库：**
- 分支 Droidian 图像存储库：[droidian-images/droidian](https://github.com/droidian-images/droidian)。
- 分支 rootfs 模板存储库（可选）：[droidian-releng/rootfs-templates](https://github.com/droidian-releng/rootfs-templates)。

**创建`community_devices.yml`：**
- 在您分支的 Droidian 图像存储库中，创建“community_devices.yml”以包含您的设备。使用“devices.yml”中的现有设备条目作为参考。
> 注意：建议使用 `community_devices.yml`，因为它使从上游存储库的合并变得更容易。

**添加设备包：**
- **使用 `pre-overlay/` 目录：** 将您的 APT 源文件添加到 Droidian 图像存储库中的 `pre-overlay/etc/apt/sources.list.d/` 中。
- **使用 Rootfs 模板（替代）：** 如果不使用 `pre-overlay/` 目录，请将 APT 源文件添加到分支 rootfs 模板中的 `apt/etc/apt/sources.list.d/` 中。更新 Droidian Images 存储库分支中的 `.gitmodules` 文件以指向您的分支 rootfs 模板。
- **使用 `apt/` 目录（替代）：** 将您的设备包文件及其发布文件添加到 Droidian Images 分支存储库中的 `apt/` 目录。如果您没有专用的 deb 包存储库，则使用此方法。

> 注意：建议托管软件包存储库，因为它可以更轻松地在设备上更新软件包。您可以在[此处](./host-package-repo.md)找到托管存储库的说明。

**触发工作流程：**
- 在 Droidian Images 分支存储库中启动工作流程以开始生成 auto night images。
