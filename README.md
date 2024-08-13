Droidian porting guide
======================

Droidian is a GNU/Linux distribution based on top of Debian for mobile devices.

The goal of Droidian is to be able to run Debian on Android phones.

This is accomplished by using well-known technologies such as [libhybris](https://github.com/libhybris/libhybris) and [halium](https://halium.org).

If your device is launched with Android 9 or above it is possible to port Droidian to it.

Legacy devices without an Android 9 port cannot be ported to Droidian. So it's either Android 9 or bust!

Translations
------------

This porting guide is also available in the following languages:
* [Chinese (中文)](https://github.com/droidian/porting-guide/tree/zh_CN)

Contents
--------

* Currently known-to-work and supported devices
  * [Droidian device page](https://devices.droidian.org)
* Porting guide
  * [Kernel compilation](./kernel-compilation.md)
  * [Tips to aid debugging](./debugging-tips.md)
  * [Rootfs creation](./rootfs-creation.md)
  * [Host package repo](./host-package-repo.md)

Getting community help
----------------------

### Search the Droidian group

The [Droidian telegram group](https://t.me/DroidianLinux/) (also bridged to [Matrix](https://matrix.to/#/%23droidian:matrix.org)) is a great resource.
It's quite possible that your issue has been already discussed and resolved.

Please use the search function rather than ask straight away. Discussing the same things at length
is boring especially when it has been already done in the past.

If you can't find an answer for your issue, try to be as detailed as possible (include a pastebin of your logs).
When someone is available, they will try to help you out.

Note that no one owes you an answer, avoid pinging people repeatedly. Avoid posting screenshots of
text messages, use a pastebin service instead.
