Droidian 移植指南
======================

Droidian 是一个基于 Debian 的 GNU/Linux 发行版，专为移动设备设计。

Droidian 的目标是使 Debian 能在 Android 手机上运行。

这一目标通过使用一些知名的技术实现，例如 [libhybris](https://github.com/libhybris/libhybris) 和 [halium](https://halium.org)。

如果你的设备是 Android 9 或更高版本发布的，那么就有可能将 Droidian 移植到该设备上。

对于没有 Android 9 移植的旧设备，不能移植到 Droidian 上。所以要么 Android 9，要么就没戏了！

翻译
------------

本移植指南还提供以下语言版本：
* [English](https://github.com/droidian/porting-guide)

内容
--------

* 当前已知的工作和支持设备
  * [Droidian 设备页面](https://devices.droidian.org)
* 移植指南
  * [内核编译](./kernel-compilation.md)
  * [调试技巧](./debugging-tips.md)
  * [Rootfs 创建](./rootfs-creation.md)
  * [软件包仓库](./host-package-repo.md)

获取社区帮助
----------------------

### 搜索 Droidian 组

[Droidian 电报组](https://t.me/DroidianLinux/)（也连接到 [Matrix](https://matrix.to/#/%23droidian:matrix.org)）是一个很好的资源。
你的问题很可能已经被讨论和解决过。

请使用搜索功能，而不是直接提问。讨论相同的内容一再重复是很无聊的，特别是当这些内容已经被讨论过时。

如果你找不到答案，请尽可能详细地描述你的问题（包括日志的 pastebin 链接）。
当有人有空时，他们会尽量帮助你。

请注意，没有人有义务回答你的问题，避免重复标记别人。避免发布文本消息的截图，改用 pastebin 服务。
