Debugging tips
==============

When porting a new device, you might (will) encounter some issues.
It's unlikely that everything will work fine at first boot, so strap in, read
this document twice and enjoy your ride.

Tips marked with **(generic rootfs only)** are useful/applicable only on
Droidian installations installed via the generic rootfs .zip flashed via recovery (note
that this document assumes TWRP as the recovery used).

Likewise, tips marked with **(fastboot only)** are useful/applicable only on
Droidian installations flashed via fastboot.

Generic tips
------------

These things are worth checking first:

### (generic rootfs only) Check size of the Droidian rootfs

Some recoveries might fail in resizing the Droidian rootfs. This will in turn
break the boot process (and eventual feature bundles installation) due to the
lack of available free space.

You can open a shell from your recovery (either via `adb shell` or with the built-in terminal) and check:

	(recovery)$ ls -lth /data/rootfs.img

if the image size is not 8GB, you can attempt resizing it manually:

	(recovery)$ e2fsck -fy /data/rootfs.img
	(recovery)$ resize2fs -f /data/rootfs.img 8G

### (generic rootfs only) Install devtools

Stable Droidian releases require the installation of the devtools feature bundle
to get useful tools to aid debugging, including an SSH server and tools required
to expose it via RNDIS.

Devtools will also force the systemd journal to be flushed to disk, so that eventual
logs might be looked for in recovery.

### (generic rootfs only) Try a nightly release

Nightly releases embed devtools on the rootfs. If you have trouble installing devtools
on a stable release, consider trying a nightly image so that you don't need to install
it separatedly.

No Droidian logo displayed or Android/Ubuntu Touch boots rather than Droidian
-----------------------------------------------------------------------------

### Double-check cmdline for the `systempart=` option

If you're re-using the same boot image from Ubuntu Touch, you might have the
`systempart=` option in the kernel cmdline.

Unlike Ubuntu Touch, Droidian doesn't use the existing system partition for the
rootfs. Thus, if you have the `systempart=` kernel option, remove it and recompile
your kernel (or otherwise remove it from the bootimage using tools like yabit or
Android Image Kitchen).

Device reboots immediately
--------------------------

### (generic rootfs only) Mask journald

Some devices have trouble with systemd-journald. You might try masking it via recovery.

Note that masking journald will disable log collection, so other issues will be
harder to debug.

	(recovery)$ mkdir /tmp/mpoint
	(recovery)$ mount /data/rootfs.img /tmp/mpoint
	(recovery)$ chroot /tmp/mpoint /bin/bash
	(recovery)$ export PATH=/usr/bin:/usr/sbin
	(recovery)$ systemctl mask systemd-journald

### Check pstore for clues

If you have pstore enabled, you might find clues in `/sys/fs/pstore` from your
recovery.

If you don't have it enabled, you should add these options to your kernel configuration:

```
CONFIG_PSTORE=y
CONFIG_PSTORE_CONSOLE=y
CONFIG_PSTORE_RAM=y
CONFIG_PSTORE_RAM_ANNOTATION_APPEND=y
```

### (generic rootfs only) Check systemd journal for clues

If you haven't masked journald, and have devtools (or a nightly image) installed, you
can check the systemd journal:

	(recovery)$ mkdir /tmp/mpoint
	(recovery)$ mount /data/rootfs.img /tmp/mpoint
	(recovery)$ chroot /tmp/mpoint /bin/bash
	(recovery)$ export PATH=/usr/bin:/usr/sbin
	(recovery)$ journalctl --no-pager

Stuck at the glowing Droidian logo, and RNDIS is not working
------------------------------------------------------------

**Before proceeding, read the previous section, especially the "Check pstore for clues"
and "Check systemd journal for clues" parts. They'll help here as well.**

If you have devtools installed (or have flashed a nightly image) but still don't
have working RNDIS, these tips might help you:

### (generic rootfs only) Mask resolved and timesyncd

Some kernels have trouble with the kernel namespaces systemd creates to run some
daemons, like timesyncd and resolved. This might make the boot process hang.

You might try masking those two services via a recovery shell:

	(recovery)$ mkdir /tmp/mpoint
	(recovery)$ mount /data/rootfs.img /tmp/mpoint
	(recovery)$ chroot /tmp/mpoint /bin/bash
	(recovery)$ export PATH=/usr/bin:/usr/sbin
	(recovery)$ systemctl mask systemd-resolved
	(recovery)$ systemctl mask systemd-timesyncd

**If this worked, re-check your kernel configuration. See the section above.**

### (generic rootfs only) Disable the Halium container

Some vendor scripts might conflict with the usb-tethering script used to set-up
the RNDIS connection.

You can disable the Halium container startup with the following:

	(recovery)$ mkdir /tmp/mpoint
	(recovery)$ mount /data/rootfs.img /tmp/mpoint
	(recovery)$ chroot /tmp/mpoint /bin/bash
	(recovery)$ export PATH=/usr/bin:/usr/sbin
	(recovery)$ nano /etc/systemd/system/lxc@android.service

Comment the ExecStartPre and ExecStart lines, add

`ExecStart=/bin/true`

save, sync and reboot

### (generic rootfs only) Setup connection via WLAN

You might try pre-configuring your WLAN device and attempt getting in via WLAN
rather than RNDIS:

	(recovery)$ mkdir /tmp/mpoint
	(recovery)$ mount /data/rootfs.img /tmp/mpoint
	(recovery)$ chroot /tmp/mpoint /bin/bash
	(recovery)$ export PATH=/usr/bin:/usr/sbin
	(recovery)$ nano /etc/network/interfaces

and put the following there:

```
auto wlan0
iface wlan0 inet dhcp
  wpa-ssid SSID
  wpa-psk PASSWORD
```

where SSID is your AP SSID, and PASSWORD is your password.

You can also use a static configuration if you prefer:

```
auto wlan0
iface wlan0 inet static
  wpa-ssid SSID
  wpa-psk PASSWORD
  address <ip>
  netmask <netmask>
  gateway <gw>
```

Reboot, and if everything went smoothly you can try getting in with `ssh droidian@<IP>`.

If you configured using DHCP, you should check among your DHCP server leases.

The default password is `1234`.

### Continue trying other stuff

If you are still unable to get a shell, you might want to continue reading
and try the other tips shown in this document. Keep in mind that you are ought
to try them via the Android recovery since you don't have SSH working. Some tips
might not apply in this case.

Stuck at the Droidian logo, but I have a shell
----------------------------------------------

**Before proceeding, read the previous section, those tips are useful for this case as well.**

**NOTE THAT EVERY COMMAND IN THIS SECTION IMPLIES ROOT PRIVILEGES UNLESS EXPLICITLY SPECIFIED**

You can start a root shell with

	(device)$ sudo -i

### Umount schedtune

Schedtune must be disabled for Droidian to work, but some device kernels complain
about that. So the workaround is to umount schedtune at boot:

	(device)# mkdir -p /etc/systemd/system/android-mount.service.d
	(device)# nano /etc/systemd/system/android-mount.service.d/10-schedtune.conf

add this there:

```
[Service]
ExecStartPre=-/usr/bin/umount -l /sys/fs/cgroup/schedtune
```

save and reboot

### Ensure the Halium container is running

You can use

	(device)# lxc-ls --fancy

to check if the Halium container is running

### Launch the Halium container manually

If the Halium container is not running, you might try launching it manually:

	(device)# lxc-start -n android --logfile=/tmp/lxclog --logpriority=DEBUG

then check `/tmp/lxclog`.

### Re-generate udev rules

If the container is running, but you still haven't booted up to UI, you might try
regenerating the udev rules (snippet from the [UBports halium-9 porting notes](https://github.com/ubports/porting-notes/wiki/Halium-9#generating-udev-rules))

	(device)# DEVICE=codename # replace with your device codename
	(device)# cat /var/lib/lxc/android/rootfs/ueventd*.rc /vendor/ueventd*.rc | grep ^/dev | sed -e 's/^\/dev\///' | awk '{printf "ACTION==\"add\", KERNEL==\"%s\", OWNER=\"%s\", GROUP=\"%s\", MODE=\"%s\"\n",$1,$3,$4,$2}' | sed -e 's/\r//' >/etc/udev/rules.d/70-$DEVICE.rules

Then reboot.

### Check if test_hwcomposer works

Trying the `test_hwcomposer` command might be an useful test to check if the Android composer works.

Note that on Halium 10/11 devices you might need to restart the Android composer service after
using a client like test_hwcomposer or phoc itself. You can kill the `android.service.composer` process, it will be
respawned by the android init automatically.

### Make the phosh service wait some seconds before start-up

Sometimes phosh might attempt its startup even when the android composer service is not
ready (even if it signals so itself). This is a bug, and you can workaround that by
making the phosh service (which starts the compositor) wait some seconds:

	(device)# mkdir -p /etc/systemd/system/phosh.service.d/
	(device)# nano /etc/systemd/system/phosh.service.d/90-wait.conf

and put this there:

```
# FIXME
[Service]
ExecStartPre=/usr/bin/sleep 5
```

Save and reboot.
