Droidian porting guide
======================

Droidian is a GNU/Linux distribution based on top of [Mobian](https://mobian-project.org), a Debian-based distribution for mobile devices.

The goal of Droidian is to be able to run Mobian on Android phones.

This is accomplished by using well-known technologies such as [libhybris](https://github.com/libhybris/libhybris) and [halium](https://halium.org).

If your device is launched with Android 9 or above it is possible to port Droidian to it.
If it already has a halium-compliant kernel of halium-9.0 and above chances are that Droidian will work without much modification.

The "System image creation" guide should only be used for legacy devices (devices released without a vendor partition) which have an Android 9 port (device tree, vendor tree and kernel source).

Legacy devices without an Android 9 port cannot be ported to Droidian. So it's either Android 9 or bust!

Any device released with Android 8.1 or later can use the generic system image provided in Droidian and skip this section.

Contents
--------

* Currently known-to-work and supported devices
  * [Droidian device page](https://devices.droidian.org)
* Porting guide
  * [Kernel compilation](./kernel-compilation.md)
  * [System image creation](./image-creation.md), skip this section if your device is treble/has a vendor partition (released with Android 8.1 or later).
  * [Tips to aid debugging](./debugging-tips.md)
  * [Rootfs creation](./rootfs-creation.md)

Getting community help
----------------------

### Search the Droidian group

The [Droidian telegram group](https://t.me/DroidianLinux/) (also bridged to [Matrix](https://matrix.to/#/%23droidian:matrix.org)) is a great resource.
It's quite possible that your issue has been already discussed and resolved.

Please use the search function rather than ask straight away. Discussing the same things at length
is boring especially when it has been already done in the past.

If you can't find an answer for your issue, try to be the more detailed possible (include a pastebin of your
logs). If someone is available it will try to help you.

Note that no one owes you an answer, avoid pinging people repeatedly. Avoid posting screenshots of
text messages, use a pastebin service instead.

