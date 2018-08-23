---
title: fwupd - Update Firmware on Linux Automatically
tags: fwupd vendor firmware service secure portal updates gentoo
---

## fwupd - Update Firmware on Linux Automatically

Recently Lenovo has started using Linux Update Firmware Service - short
[LVFS](https://fwupd.org/). This is very good news! This allows you to update
your BIOS without an existing Windows installation. Of course there are other
ways but this is often quite a hassle. Due to lack of documentation on how to get
it to work in Gentoo, you can follow these steps.

[Supported devices](https://fwupd.org/lvfs/devicelist)

First of all have a short look at the [basic usage
flow](https://github.com/hughsie/fwupd#basic-usage-flow-command-line). After
emerging `sys-apps/fwupd` (pay attention to the gpg and uefi USE flag) and
starting it's service with `rc-service fwupd start` its most likely that
`fwupdmgr get-devices` returns "No detected devices." 

You probably want to activate the `fwupd` service permanently with `rc-update add
fwupd default`

According to the [Arch
Wiki](https://wiki.archlinux.org/index.php/fwupd#Setup_for_UEFI_BIOS_upgrade)
for a BIOS upgrade make sure you booted in UEFI mode, make sure your UEFI
variables are accessible and mount your EFI system partition properly.

### Enable EFI Variable Support

Enable EFI Variable Support via sysfs (CONFIG_EFI_VARS) so that the efivars can
be mounted.

```
Firmware Drivers  --->
    EFI (Extensible Firmware Interface) Support  --->
        <*> EFI Variable Support via sysfs
```

Recompile the kernel and boot your system with the new kernel.

### Make UEFI Variables Accessible

Your efivars are accessible when `/sys/firmware/efi/efivars` is mounted `rw`.
Check it with `mount | grep efivars`. Get write access by remounting `mount -o
rw,remount /sys/firmware/efi/efivars`.

If it doesn't work now we need to take a closer look.

### Examine the Daemon

We are now going to stop the `fwupd` service and execute the binary on our own.
Pay attention to errors!

Check which binary the service executes `cat /etc/init.d/fwupd`. Something like
this will show up:

```
#!/sbin/openrc-run
# Copyright 1999-2018 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

description="Firmware update daemon"
pidfile="/var/run/fwupd.pid"
command="/usr/libexec/fwupd/fwupd"
command_background="true"

depend() {
  need dbus
  before xdm
}
```

Stop the service `rc-service fwupd stop` and start the daemon on the console
`/usr/libexec/fwupd/fwupd` (path from the above file).

### Unable to determine EFI system partition location

You need to edit your `/etc/fwupd/uefi.conf` so that the path is correct.

### Boot Order Lock

I had no problem with that but please see this
[comment](https://www.reddit.com/r/thinkpad/comments/949a1h/lenovo_has_started_using_lvfs/e3jcnma):
"Also note that if you turn on Boot Order Lock in BIOS setup, this won't work
and there'll be no explanation why. It will just reboot and do nothing at all.
Took me a couple of hours last week."
