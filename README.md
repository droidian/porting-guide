hybris-mobian porting guide
===========================

hybris-mobian is a GNU/Linux distribution based on top of [Mobian](https://mobian-project.org),
a Debian-based distribution for mobile devices.

The goal of hybris-mobian is to be able to run Mobian on Android phones.

This is accomplished by using well-known technologies such as [libhybris](https://github.com/libhybris/libhybris)
and [halium](https://halium.org).

If your device supports `halium-9.0`, chances are that it can run hybris-mobian easily.

Contents
--------

* Currently known-to-work and supported devices
* Porting guide
  * [Kernel compilation](./kernel-compilation.md)
  * Image creation
  * Tips to aid debugging
