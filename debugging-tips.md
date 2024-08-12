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

**在继续之前，请阅读前一部分，这些提示可能也会有用。**

**请注意，本节中的每个命令都需要 root 权限，除非明确说明**

您可以使用以下命令启动一个 root shell：

    (设备)$ sudo -i

### USB 网络共享

要将主机的网络连接与设备共享，您可以在主机上使用以下命令：

    (主机)$ sudo sysctl net.ipv4.ip_forward=1
    (主机)$ sudo iptables -t nat -A POSTROUTING -o $INTERNET -j MASQUERADE
    (主机)$ sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    (主机)$ sudo iptables -A FORWARD -i $USB -o $INTERNET -j ACCEPT

其中 `$USB` 是连接到设备的接口，通常是 `usb0`。

`$INTERNET` 是连接到互联网的接口。

然后在设备上运行以下命令：

    (设备)$ sudo ip route add default via 10.15.19.100

IP 地址 `10.15.19.100` 是您的主机地址。在运行命令之前请仔细检查。

现在要访问互联网，请将 `/etc/resolv.conf` 的内容替换为 `nameserver 8.8.8.8`。

### 卸载 schedtune

要使 Droidian 正常工作，必须禁用 schedtune，但有些设备内核会对此发出警告。解决方法是在启动时卸载 schedtune：

    (设备)# mkdir -p /etc/systemd/system/android-mount.service.d
    (设备)# nano /etc/systemd/system/android-mount.service.d/10-schedtune.conf

在文件中添加以下内容：

```
[Service]
ExecStartPre=-/usr/bin/umount -l /sys/fs/cgroup/schedtune
```

保存并重启设备。

### 确保 Halium 容器正在运行

您可以使用以下命令检查 Halium 容器是否正在运行：

    (设备)# lxc-ls --fancy

### 手动启动 Halium 容器

如果 Halium 容器没有运行，您可以尝试手动启动它：

    (设备)# lxc-start -n android --logfile=/tmp/lxclog --logpriority=DEBUG

然后检查 `/tmp/lxclog`。

### 重新生成 udev 规则

如果容器正在运行，但您仍未启动 UI，您可以尝试重新生成 udev 规则：

    (设备)# DEVICE=codename # 替换为您的设备代号
    (设备)# cat /var/lib/lxc/android/rootfs/ueventd*.rc /var/lib/lxc/android/rootfs/system/etc/ueventd*.rc /vendor/ueventd*.rc /var/lib/lxc/android/rootfs/vendor/etc/ueventd*.rc | grep ^/dev | sed -e 's/^\/dev\///' | awk '{printf "ACTION==\"add\", KERNEL==\"%s\", OWNER=\"%s\", GROUP=\"%s\", MODE=\"%s\"\n",$1,$3,$4,$2}' | sed -e 's/\r//' >/etc/udev/rules.d/70-$DEVICE.rules

然后重启设备。

### 检查 `test_hwcomposer` 是否正常工作

尝试使用 `test_hwcomposer` 命令可能是检查 Android composer 是否正常工作的一个有效测试。

注意，在 Halium 10/11 设备上，您可能需要在使用如 `test_hwcomposer` 或 `phoc` 客户端后重启 Android composer 服务。您可以杀死 `android.service.composer` 进程，它将由 android init 自动重启。

### 使 Phosh 服务在启动前等待几秒钟

有时 Phosh 可能会尝试启动，即使 Android composer 服务尚未准备好（即使它自己发出了信号）。

这是一个 bug，您可以通过让 Phosh 服务（启动 compositor）等待几秒钟来解决：

    (设备)# mkdir -p /etc/systemd/system/phosh.service.d/
    (设备)# nano /etc/systemd/system/phosh.service.d/90-wait.conf

并在文件中添加以下内容：

```
# FIXME
[Service]
ExecStartPre=/usr/bin/sleep 5
```

保存并重启设备。

### Vendor 分区未挂载

可能 Vendor 分区未被 init 挂载，导致 Halium 容器无法启动许多服务。

首先检查 vendor 镜像是否已挂载：

    (设备)# ls /vendor

如果为空，则说明 init 未挂载。尝试通过查找 `/dev/disk/by-partlabel` 和 `/dev/block/bootdevice/by-name` 中的分区来手动挂载它。

如果未找到，您需要在 `/dev` 中搜索您的 vendor 分区。

然后，为了测试，在 `/var/lib/lxc/android/pre-start.sh` 中的 `# Halium 9` 部分下添加以下内容：

```
mkdir -p /var/lib/lxc/android/rootfs/vendor
mount /dev/mmcblk0pYOURVENDOR /vendor
mount --bind /vendor /var/lib/lxc/android/rootfs/vendor
```

然后重启设备。

稍后请务必回到此问题以找到合适的解决方案。

### lxc@android 和 phosh 已启动但没有输出

如果 phosh 和 lxc@android 已经启动并且 udev 规则也已到位，但屏幕上没有输出，可能是 vndservicemanager 崩溃了。

在某些 Halium 10 设备上，可能需要使用修补版本的 vndservicemanager 以使其启动。可以在 [这里](https://github.com/droidian-devices/adaptation-droidian-starqlte/blob/droidian/usr/lib/droid-vendor-overlay/bin/vndservicemanager) 找到修补版本的 vndservicemanager。

    (设备)# mkdir -p /usr/lib/droid-vendor-overlay/bin/
    (设备)# cp vndservicemanager /usr/lib/droid-vendor-overlay/bin/

重启设备。

### 长时间启动

如果遇到长时间启动的问题，您可以尝试检查内核日志，查看哪个进程阻碍了系统启动：

    (设备)# dmesg

以找出导致启动问题的部分：

    (设备)# systemd-analyze

### 系统启动后的小贴士

#### 屏幕亮度

在某些 Qualcomm 设备上，屏幕亮度总是通过 hwcomposer 在启动时设置为 0。为了解决此问题，可以在每次启动时将亮度设置为最大值：

    (设备)$ echo 2047 > /sys/class/leds/lcd-backlight/brightness

也可以像 [这个服务文件](https://github.com/droidian-devices/adaptation-droidian-lavender/blob/droidian/debian/adaptation-lavender-configs.brightness.service) 一样，将其设置为在启动时自动启动。

#### 下拉菜单中的亮度调节

在某些设备上，vendor 在启动时为亮度 sysfs 节点设置了错误的权限，因此 Phosh 不能访问和修改它。

可以使用类似 [这个服务](https://github.com/droidian-devices/adaptation-droidian-onclite/blob/droidian/debian/adaptation-onclite-configs.brightnessperm.service) 的服务来设置正确的权限，或者至少让用户 droidian 可以访问 sysfs 节点。

确保根据需要替换亮度节点。

#### Phosh 缩放

Phosh 可能设置了错误的缩放比例。应创建 `phoc.ini` 来调整它：

    (设备)# mkdir -p /etc/phosh/
    (设备)# nano phoc.ini

并放入 [phoc.ini](https://github.com/droidian-devices/adaptation-droidian-miatoll/blob/droidian/etc/phosh/phoc.ini)

`output:HWCOMPOSER-1` 的值可以根据需要进行调整。

#### Vendor 分区覆盖

要覆盖 vendor 分区上的文件，可以使用 `droid-vendor-overlay` 目录：

    (设备)# mkdir -p /usr/lib/droid-vendor-overlay

然后可以在此处添加您的文件。可以参考 [这个](https://github.com/droidian-devices/adaptation-fxtec-pro1x/tree/bookworm/sparse/usr/lib/droid-vendor-overlay)。

请注意，目录结构非常重要。

#### 系统分区覆盖

要覆盖系统分区上的文件，可以使用 `droid-system-overlay` 目录：

    (设备)# mkdir -p /usr/lib/droid-system-overlay

然后可以在此处添加您的文件。可以参考 [这个](https://github.com/droidian-devices/adaptation-fxtec-pro1x/tree/droidian/sparse/usr/lib/droid-system-overlay)。

请注意，目录结构非常重要。

#### 蓝牙崩溃

蓝牙可能由于缺少 MAC 地址而崩溃。可以使用以下 hack 使蓝牙服务忽略此问题：

    (设备)# mkdir -p /var/lib/bluetooth/
    (设备)# touch /var/lib/bluetooth/board-address

为了找到合适的解决方案，应该创建一个 `droid-get-bt-address.sh` 脚本来获取正确的地址。

可以在

 [这里](https://github.com/droidian-devices/adaptation-google-sargo/blob/droidian/usr/bin/droid/droid-get-bt-address.sh) 找到 Qualcomm 设备的示例解决方案，在 [这里](https://github.com/droidian-devices/adaptation-droidian-oneplus3/blob/droidian/usr/bin/droid/droid-get-bt-address.sh) 找到 MediaTek 设备的示例解决方案。

#### 设置中没有蓝牙选项

蓝牙可能由于各种原因未出现在 `gnome-control-center` 中。

可以使用 `bluetoothctl` 或 `blueman` 进行测试：

    (设备)$ bluetoothctl
    [bluetooth]# scan on

或者安装 `blueman`：

    (设备)# apt install blueman

如果设置中显示了蓝牙选项但没有设备，可以安装新的包 `btscanner-gcc`：

    (设备)# apt install btscanner-gcc

然后重启设备，蓝牙应该会出现。

#### 音频调节无效

在 MediaTek 设备上，pulseaudio 需要一个自定义配置文件，可以在 [这里](https://github.com/droidian-devices/adaptation-droidian-angelica/blob/droidian/etc/pulse/arm_droid_card_custom.pa) 找到。

#### 自定义主机名

要在启动时设置自定义主机名，可以创建一个首选主机名文件：

    (设备)# mkdir -p /usr/lib/droidian/device/
    (设备)# nano preferred-hostname

并填写您的设备型号或代号，不要有空格。

#### 屏幕上的光标

一些设备有一些 Droidian 未使用的 HID 接口。这些接口可能被注册为键盘或鼠标等设备。

因此，您可能会在屏幕上看到光标，但没有明显的原因。

要获取输入设备的信息，可以安装并使用 `libinput-tools`：

    (设备)# apt install libinput-tools

列出所有设备：

    (设备)# libinput list-devices

监视输入事件：

    (设备)# libinput debug-events

要隐藏这个事件节点，可以在 `/etc/udev/rules.d/71-hide.rules` 中添加一个 udev 规则：

`ACTION=="add|change", KERNEL=="event1", OWNER="root", GROUP="system", MODE="0666", ENV{LIBINPUT_IGNORE_DEVICE}="1"`

确保根据您的输入设备调整事件节点，并重启设备。

#### 加密支持

要测试加密，首先在 `/usr/lib/droidian/device/encryption-supported` 中创建一个空文件，然后打开加密应用程序：

    (设备)# mkdir -p /usr/lib/droidian/device/
    (设备)# touch /usr/lib/droidian/device/encryption-supported

如果启用加密后设备可以顺利解锁，则保留该文件。否则，请删除它。

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
