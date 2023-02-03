Droidian porting guide
===========================

Droidian is a GNU/Linux distribution based on top of [Mobian](https://mobian-project.org),
a Debian-based distribution for mobile devices.

The goal of Droidian is to be able to run Mobian on Android phones.

This is accomplished by using well-known technologies such as [libhybris](https://github.com/libhybris/libhybris) and [halium](https://halium.org).

If your device is launched with Android 9 or above it is possible to port Droidian to it.
If it already has a halium-compliant kernel of halium-9.0 and above chances are that Droidian will work without much modification.

Contents
--------

* Currently known-to-work and supported devices
  * [Droidian device page](https://devices.droidian.org)
* Porting guide
  * [Kernel compilation](./kernel-compilation.md)
  * [Image creation](./image-creation.md)
  * [Tips to aid debugging](./debugging-tips.md)
