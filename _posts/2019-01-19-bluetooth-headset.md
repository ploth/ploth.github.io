---
title: Set Up Bluetooth Headset on Gentoo Including Headset Control Interface
tags: bluetooth bluez headset microphone speaker touch interface d-feet dbus sony wh-1000xm3
---

## Set Up Bluetooth Headset on Gentoo Including Headset Control Interface

The following setup allows you to easily connect your Bluetooth headset to
Gentoo. With a small script you can switch between the internal speakers and the
headset within milliseconds. You can use the buttons on the headset to control
your computer or music. Even the touch interface of the Sony WH-1000XM3 is
working.

### Pulseaudio

`media-sound/pulseaudio` is required.

### udev

{% highlight %}
General setup  --->
    [*] Configure standard kernel features (expert users)  --->
        [ ] Enable deprecated sysfs features to support old userspace tools
        [*] Enable signalfd() system call
Enable the block layer  --->
    [*] Block layer SG support v4
Networking support  --->
    Networking options  --->
        <*> Unix domain sockets
Device Drivers  --->
    Generic Driver Options  --->
        ()  path to uevent helper
        [*] Maintain a devtmpfs filesystem to mount at /dev
    < > ATA/ATAPI/MFM/RLL support (DEPRECATED)  --->
    Input device support  ---> 
        [*] Generic input layer (needed for keyboard, mouse, ...)
            [*] Miscellaneous devices  --->
                <*> User level driver support
File systems  --->
    [*] Inotify support for userspace
    Pseudo filesystems --->
        [*] /proc file system support
        [*] sysfs file system support
{% endhighlight %}

{% highlight Shell %}
emerge --ask --oneshot sys-fs/eudev
rc-update add udev sysinit
{% endhighlight %}

### Bluetooth

{% highlight %}
[*] Networking support --->
      <M>   Bluetooth subsystem support --->
              [*]   Bluetooth Classic (BR/EDR) features
              <*>     RFCOMM protocol support
              [ ]       RFCOMM TTY support
              < >     BNEP protocol support
              [ ]       Multicast filter support
              [ ]       Protocol filter support
              <*>     HIDP protocol support
              [*]     Bluetooth High Speed (HS) features
              [*]   Bluetooth Low Energy (LE) features
                    Bluetooth device drivers --->
                      <M> HCI USB driver
                      <M> HCI UART driver
      <*>   RF switch subsystem support --->
    Device Drivers --->
          HID support --->
            <*>   User-space I/O driver support for HID subsystem
{% endhighlight %}

{% highlight Shell %}
emerge -av --noreplace net-wireless/bluez
gpasswd -a <user> plugdev
rc-service bluetooth start
rc-update add bluetooth default
{% endhighlight %}

#### Device Pairing

[Arch Wiki](https://wiki.archlinux.org/index.php/bluetooth#Pairing)

### Bluetooth Input Devices

{% highlight %}
Device Drivers  --->
    [*] HID Devices  --->

          You may need some special driver for your input device:
          Special HID drivers  --->
                <*> ...

[*] Networking support  --->
    <*>   Bluetooth subsystem support  --->
          <*>   L2CAP protocol support
          <*>   HIDP protocol support<Paste>
{% endhighlight %}

Enable HID protocol handling in userspace input profile by adding
`UserspaceHID=true` to `/etc/bluetooth/input.conf`.

### Button Interface

You should now be able to receive `XF86AudioPrev`, `XF86AudioNext`,
`XF86AudioPlay`, `XF86AudioPause` key presses or whatever your bluetooth
headset is able to send. `xev` helps you with this task.

#### Awesome-wm

With `playerctl` installed you can register the following key objects inside
your `rc.lua`.

{% highlight linenos %}
awful.key({                   }, "XF86AudioPrev",
  function ()
	awful.util.spawn("playerctl previous")
  end),
awful.key({                   }, "XF86AudioPlay",
  function ()
	awful.util.spawn("playerctl play")
  end),
awful.key({                   }, "XF86AudioPause",
  function ()
	awful.util.spawn("playerctl pause")
  end),
awful.key({                   }, "XF86AudioNext",
  function ()
	awful.util.spawn("playerctl next")
  end)
{% endhighlight %}

### Debugging

#### Bluetooth

`rfkill` is usefull if your computer's bluetooth device is soft or hard blocked.

#### d-bus

`d-feet` shows registered bluetooth devices with their media control features.

#### Pulseaudio

`pavucontrol` helps visualizing the current audio output configuration.

## Acknowledgement

Thanks to [@nagua](https://github.com/nagua/) for the help with the input devices.
