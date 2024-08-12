调试提示
==============

一组可能帮助你调试 Droidian 移植问题的提示和修复方法。

目录
-----------------

* [总结](#总结)
* [启动前提示](#启动前提示)
* [系统启动到 shell 时的提示](#系统启动到-shell-时的提示)
* [系统启动到 Phosh 时的提示](#系统启动到-phosh-时的提示)

总结
-------

在移植新设备时，你可能会遇到一些问题。系统第一次启动时一切顺利的可能性很小，所以请耐心阅读本文档两遍，享受过程。

注意：本文档假设你使用的是 TWRP 作为恢复工具。

启动前提示
-------------

首先检查以下内容：

### 检查 Droidian 根文件系统的大小

有些恢复工具可能无法调整 Droidian 根文件系统的大小。这会导致启动过程（以及可能的功能包安装）失败，因为可用的空闲空间不足。

你可以从恢复工具中打开一个 shell（通过 `adb shell` 或内置终端），并检查：

    (recovery)$ ls -lth /data/rootfs.img

如果镜像大小不是 8GB，你可以尝试手动调整其大小：

    (recovery)$ e2fsck -fy /data/rootfs.img
    (recovery)$ resize2fs -f /data/rootfs.img 8G

### 开发工具

快照版 Droidian 需要安装 devtools 功能包以获取有用的调试工具，包括 SSH 服务器和通过 RNDIS 暴露它的工具。

devtools 还会强制 systemd 日志被刷新到磁盘，以便在恢复模式下查找日志。

每夜版已经在根文件系统中嵌入了 devtools。如果你在快照版中安装 devtools 有困难，可以考虑尝试每夜版镜像，这样你就不需要单独安装它。

### 双重检查 cmdline 中的 `systempart=` 选项

如果你在重用 Ubuntu Touch 的相同启动镜像，可能会在内核 cmdline 中找到 `systempart=` 选项。这可能会导致 Android/Ubuntu Touch 启动，而不是 Droidian。

与 Ubuntu Touch 不同，Droidian 不使用现有的系统分区作为根文件系统。因此，如果你有 `systempart=` 内核选项，请将其删除并重新编译内核（或使用 yabit 或 Android Image Kitchen 等工具从启动镜像中删除它）。

如果设备立即重启，你应该尝试以下几个 systemd 部分。

### 启动到 fastboot

由于某些未知原因，一些设备在使用 arm64 ramdisk 时无法启动（yggdrasil 和 sargo 更容易出现此问题）。

要为你的设备使用 armhf ramdisk，应该使用 [这个](https://github.com/droidian-devices/linux-android-google-sargo/blob/droidian/debian/rules) `debian/rules` 文件，而不是默认使用的文件，并将 `linux-initramfs-halium-generic:armhf` 添加到 `debian/kernel-info.mk` 中，然后重新生成 `debian/control` 并重建内核。

### initramfs 找不到 userdata 分区

如果设备在 initramfs 中卡住（你会在主机的 dmesg 中看到“Failed to boot”），一种可能性是 initramfs 找不到 userdata 分区。

要修复此问题，可以在 cmdline 中添加 `datapart=`。

datapart 的值应该是 `/dev/disk/by-partlabel/userdata` 或该分区的确切标签和编号，如 `/dev/sda23`（这取决于设备）。

### initramfs 拒绝在目标文件系统上查找 /init

Halium 要求 `console=` 为控制台设备。对于某些设备，从 stock 启动镜像中提取的默认 cmdline 可能包含与 halium 期望的冲突的 console= 启动参数，在这种情况下，halium 将拒绝启动 /init。如果你的设备在启动内核后立即重启，而启动 cmdline 中有类似 "console=ttyMSM0,115200n8" 的内容，请将其替换为 "console=tty0" 并重新编译内核。

### 屏蔽 journald

一些设备（尤其是 Exynos 9810 和 Exynos 9820 设备）在使用 systemd-journald 时遇到问题。你可以尝试通过恢复模式屏蔽它。

注意：屏蔽 journald 将禁用日志收集，因此其他问题将更难调试。

    (recovery)$ mkdir /tmp/mpoint
    (recovery)$ mount /data/rootfs.img /tmp/mpoint
    (recovery)$ chroot /tmp/mpoint /bin/bash
    (recovery)$ export PATH=/usr/bin:/usr/sbin
    (recovery)$ systemctl mask systemd-journald

### 检查 systemd 日志以获取线索

如果你没有屏蔽 journald，并且安装了 devtools（或每夜版镜像），你可以检查 systemd 日志：

    (recovery)$ mkdir /tmp/mpoint
    (recovery)$ mount /data/rootfs.img /tmp/mpoint
    (recovery)$ chroot /tmp/mpoint /bin/bash
    (recovery)$ export PATH=/usr/bin:/usr/sbin
    (recovery)$ journalctl --no-pager

如果你卡在发光的 Droidian 标志上，并且 RNDIS 无法工作，你应该检查你的内核配置，确保 `CONFIG_USB_CONFIGFS_RNDIS` 被启用。

还应注意，你应该检查 [内核编译](./kernel-compilation.md) 指南中的 pstore 部分。

如果你已安装 devtools（或已刷写每夜版镜像），但 RNDIS 仍然无法工作，这些提示可能会帮助你：

### 屏蔽 resolved 和 timesyncd

一些内核（已知 Exynos 内核有此问题）在 systemd 创建的内核命名空间中运行某些守护进程（如 timesyncd 和 resolved）时会遇到问题。这可能导致启动过程挂起。

你可以尝试通过恢复 shell 屏蔽这两个服务：

    (recovery)$ mkdir /tmp/mpoint
    (recovery)$ mount /data/rootfs.img /tmp/mpoint
    (recovery)$ chroot /tmp/mpoint /bin/bash
    (recovery)$ export PATH=/usr/bin:/usr/sbin
    (recovery)$ systemctl mask systemd-resolved
    (recovery)$ systemctl mask systemd-timesyncd

### 禁用 Halium 容器

一些厂商脚本可能与用于设置 RNDIS 连接的 usb-tethering 脚本发生冲突。

你可以通过以下方法禁用 Halium 容器启动：

    (recovery)$ mkdir /tmp/mpoint
    (recovery)$ mount /data/rootfs.img /tmp/mpoint
    (recovery)$ chroot /tmp/mpoint /bin/bash
    (recovery)$ export PATH=/usr/bin:/usr/sbin
    (recovery)$ nano /etc/systemd/system/lxc@android.service

注释掉 ExecStartPre 和 ExecStart 行，添加

`ExecStart=/bin/true`

保存、同步并重启。

### 通过 WLAN 设置连接

你可以尝试预配置 WLAN 设备，并尝试通过 WLAN 连接而不是 RNDIS：

    (recovery)$ mkdir /tmp/mpoint
    (recovery)$ mount /data/rootfs.img /tmp/mpoint
    (recovery)$ chroot /tmp/mpoint /bin/bash
    (recovery)$ export PATH=/usr/bin:/usr/sbin
    (recovery)$ nano /etc/network/interfaces

并在那里放入以下内容：

```
auto wlan0
iface wlan0 inet dhcp
  wpa-ssid SSID
  wpa-psk PASSWORD
```

其中 SSID 是你的 AP SSID，PASSWORD 是你的密码。

如果你更喜欢使用静态配置：

```
auto wlan0
iface wlan0 inet static
  wpa-ssid SSID
  wpa-psk PASSWORD
  address <ip>
  netmask <netmask>
  gateway <gw>
```

重启后，如果一切顺利，你可以尝试通过 `ssh droidian@$IP` 进行访问。

如果你使用了 DHCP 配置，你应该在 DHCP 服务器租约中检查。

默认密码是 `1234`。

### ssh 显示系统仍在启动

为了让 ssh 忽略 systemd 的抱怨，可以在恢复模式下添加 [这个服务文件](https://github.com/droidian-devices/adaptation-droidian-angelica/blob/droidian/debian/adaptation-angelica-configs.ssh-fix.service)

    (recovery)$ mkdir /tmp/mpoint
    (recovery)$ mount /data/rootfs.img /tmp/mpoint
    (recovery)$ chroot /tmp/mpoint /bin/bash
    (recovery)$ export PATH=/usr/bin:/usr/sbin
    (recovery)$ nano /etc/systemd/system

将服务文件的内容放入其中

    (recovery)$ systemctl enable ssh-fix

系统启动到 shell 时的提示
---------------------------------------------

**在继续之前，请阅读上一节，这些提示也可能有用。**

**注意：

确保你的系统分区大小足够大。如果系统分区太小，可能会出现各种各样的问题。**

### 挂载用户数据分区

在 chroot 环境中，用户数据分区可能没有自动挂载。检查并手动挂载：

    (recovery)$ mkdir /tmp/mpoint
    (recovery)$ mount /data/rootfs.img /tmp/mpoint
    (recovery)$ chroot /tmp/mpoint /bin/bash
    (chroot)$ mkdir /data
    (chroot)$ mount /dev/block/bootdevice/by-name/userdata /data

检查数据是否在那里。你也可以尝试手动挂载：

    (chroot)$ mount -t ext4 /dev/block/bootdevice/by-name/userdata /data

### root 分区损坏

如果 root 分区损坏（文件系统损坏），你可能会遇到启动问题。你可以尝试修复文件系统：

    (recovery)$ mkdir /tmp/mpoint
    (recovery)$ mount /data/rootfs.img /tmp/mpoint
    (recovery)$ chroot /tmp/mpoint /bin/bash
    (chroot)$ e2fsck -fy /dev/sdaX

### 日志级别

确保你有足够的日志级别来获取有关挂起或崩溃的信息。如果日志级别不够高，你可能无法得到有用的调试信息。

    (chroot)$ nano /etc/systemd/system.conf

设置：

```
LogLevel=debug
```

### 升级到最新的 Halium 镜像

有时，升级到最新的 Halium 镜像可能会修复某些启动问题。请参考 Halium 的官方文档以获取更多信息。

系统启动到 Phosh 时的提示
---------------------------

**在继续之前，请阅读前面的部分，这些提示也可能有用。**

### Phosh 屏幕闪烁或无法启动

有时，Phosh 可能会闪烁或无法启动。你可以尝试调整一些设置：

1. **检查 `phosh` 的日志**：
   - 进入到 chroot 环境，检查 `/var/log/syslog` 或使用 `journalctl` 查看 Phosh 的日志。

2. **检查 Phosh 配置**：
   - 确保你的 `/etc/phosh` 配置文件正确。

3. **更新 Phosh**：
   - 有时，Phosh 的更新可以解决显示问题。尝试运行以下命令更新：

     ```
     (chroot)$ apt update
     (chroot)$ apt upgrade phosh
     ```

4. **屏幕驱动问题**：
   - 检查是否存在屏幕驱动或显卡问题，确保所有驱动程序已正确安装。

5. **检查 Wayland 设置**：
   - Phosh 使用 Wayland 作为显示服务器，确保 Wayland 的设置正确。

### 网络和蓝牙问题

确保你的网络和蓝牙适配器正常工作。你可以通过以下步骤进行检查：

1. **检查网络连接**：
   - 确保 Wi-Fi 或以太网连接正常工作。

2. **检查蓝牙状态**：
   - 确保蓝牙服务正常运行，可以使用 `systemctl status bluetooth` 查看服务状态。

3. **检查网络配置**：
   - 进入到 chroot 环境，检查 `/etc/network/interfaces` 或其他网络配置文件，确保配置正确。

希望这些提示能够帮助你解决 Droidian 移植中的问题。如果遇到其他问题，请随时提供详细信息以便进一步帮助。
