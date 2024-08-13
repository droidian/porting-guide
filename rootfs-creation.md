### 根文件系统 (rootfs) 创建
===============

为了使安装过程更简单和一致，我们会构建特定于设备的 rootfs 镜像。

目录
-----------------

* [总结](#summary)
* [先决条件](#prerequisites)
* [包创建](#package-creation)
* [构建 rootfs](#building-the-rootfs)
* [自动化夜间镜像](#automating-nightly-images)

总结
-------

在调试完成后，您应该开始为您的设备构建一个特定的 rootfs，以使安装过程更加一致和用户友好。

这个过程涉及创建一个适配包和一个 apt 仓库。

适配包是一个 Debian 包，包含所有设备特定的配置、定制的二进制文件和设备正常运行所需的脚本。

为了能够进行 OTA 更新，您必须设置一个 apt 仓库，可以托管在像 GitHub 这样的平台上或在服务器上。

创建 apt 仓库的过程超出了本教程的范围。

先决条件
-------------

* 设备特定文件
* Docker

依赖项
------------

在您的主机上需要安装 `dpkg-dev`、`gpg`、`git` 和 `apt-utils`。

```bash
(host)# apt update && apt install dpkg-dev gpg apt-utils
```

包创建
----------------

1. **克隆 Droidian 构建工具仓库**

   ```bash
   (host)$ git clone https://github.com/droidian-releng/droidian-build-tools/ && cd droidian-build-tools/bin
   ```

2. **为您的设备创建模板**
   替换 `vendor`、`codename`、`ARCH` 和 `APIVER` 为您的设备的详细信息。

   ```bash
   (host)$ ./droidian-new-device -v vendor -n codename -c ARCH -a APIVER -r phone -d droidian
   ```

   - `APIVER`: Android API 版本（例如，28、29、30）。
   - `ARCH`: 架构（例如，arm64、armhf、amd64）。

3. **初始化 QEMU 以支持 Docker**

   ```bash
   (host)$ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
   ```

4. **准备设备特定文件**
   将这些文件放置在 `sparse/` 目录中的 `adaption-vendor-codename` 目录下。

5. **设置您的仓库 URL**
   编辑 `sparse/usr/lib/adaptation-vendor-model/sources.list.d/community-vendor-device.list` 文件以包含您的仓库 URL。

   ```plaintext
   deb [signed-by=/usr/share/keyrings/codename.gpg] https://YOURREPO.TLD bookworm main
   ```

6. **复制您的 GPG 文件**
   将您的 GPG 密钥放到 `sparse/usr/share/keyrings/codename.gpg` 中。

7. **构建 Debian 包**

   ```bash
   (host)$ ~/droidian-build-tools/bin/droidian-build-package
   ```

8. **上传包文件**
   将 `~/droidian-build-tools/bin/droidian/vendor/codename/droidian/apt` 中的文件复制到您的仓库中。

9. **创建可刷写的包**
   在您的 Droidian 设备上：

   ```bash
   (device)$ apt update
   (device)$ package-sideload-create adaptation-droidian-codename.zip adaptation-vendor-codename adaptation-vendor-codename-configs
   ```

10. **提交更改**
    确保您的仓库更改已提交到 Git 仓库。

---

### 构建 rootfs

1. **添加您的内核包**
   将您在内核编译过程中构建的 `linux-*.deb` 包添加到 `community_devices.yml` 中，或作为依赖项添加到 `debian/control` 中。

2. **拉取 Rootfs Builder Docker 镜像**

   ```bash
   (host)$ docker pull quay.io/droidian/rootfs-builder:current-amd64
   ```

3. **使用 Docker 构建 Rootfs**

   ```bash
   (host)$ cd ~/droidian-build-tools/droidian/vendor/codename/droidian
   (host)$ mkdir images
   (host)$ docker run --privileged -v $PWD/images:/buildd/out -v /dev:/host-dev -v /sys/fs/cgroup:/sys/fs/cgroup -v $PWD:/buildd/sources --security-opt seccomp:unconfined quay.io/droidian/rootfs-builder:current-amd64 /bin/sh -c 'cd /buildd/sources; DROIDIAN_VERSION="nightly" ./generate_device_recipe.py vendor_codename ARCH phosh phone APIVER && debos --disable-fakemachine generated/droidian.yaml'
   ```

4. **定位 Rootfs 镜像**
   最终镜像应位于 `~/droidian-build-tools/droidian/vendor/codename/packages/adaptation-vendor-codename/droidian/images/` 中。

---

### 自动化夜间镜像

要使用 GitHub Actions 自动化生成夜间构建：

1. **分叉仓库**
   - 分叉 Droidian Images 仓库: [droidian-images/droidian](https://github.com/droidian-images/droidian)。
   - 分叉 rootfs 模板仓库（可选）: [droidian-releng/rootfs-templates](https://github.com/droidian-releng/rootfs-templates)。

2. **创建 `community_devices.yml`**
   在您的分叉的 Droidian Images 仓库中创建 `community_devices.yml` 文件以包含您的设备。

3. **添加设备包**
   - **使用 `pre-overlay/` 目录**: 将 APT 源文件添加到 `pre-overlay/etc/apt/sources.list.d/` 中。
   - **使用 Rootfs 模板（替代方案）**: 更新 `.gitmodules` 文件以指向您的分叉 rootfs 模板。
   - **使用 `apt/` 目录（替代方案）**: 将您的设备包和发行文件添加到 Droidian Images 分叉仓库的 `apt/` 目录中。

4. **触发工作流**
   在您的 GitHub Actions 仓库中启动工作流以开始生成夜间镜像。

---

通过遵循这些步骤，您将能够为 Droidian 创建和自动化设备特定的 rootfs 镜像。如果遇到问题，请确保所有路径和配置正确，并参考详细的文档或社区支持。
