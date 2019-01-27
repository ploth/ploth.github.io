---
title: Set Up Bluetooth Headset on Gentoo Including Headset Control Interface
tags: bluetooth bluez headset microphone speaker touch interface d-feet dbus sony wh-1000xm3 button interface kernel
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

#### Kernel Configuration

{% highlight Text %}
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

#### Setup

{% highlight Shell %}
emerge --ask --oneshot sys-fs/eudev
rc-update add udev sysinit
{% endhighlight %}

### Bluetooth

#### Kernel Configuration

{% highlight Text %}
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

#### Setup

{% highlight Shell %}
emerge -av --noreplace net-wireless/bluez
gpasswd -a <user> plugdev
rc-service bluetooth start
rc-update add bluetooth default
{% endhighlight %}

#### Device Pairing

[Arch Wiki](https://wiki.archlinux.org/index.php/bluetooth#Pairing)

### Bluetooth Input Devices

#### Kernel Configuration

{% highlight Text %}
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

#### Setup

Enable HID protocol handling in userspace input profile by adding
`UserspaceHID=true` to `/etc/bluetooth/input.conf`.

### Speaker Control Script

Assuming you've done the pairing process already the following script connects
computer and bluetooth headset by calling `s hs` (hs stands for headset). With
`s is` you can switch to your internal speakers. `s switch` toggles between
internal speakers and headset. Sometimes there a clients which doesn't like to
get moved to another output device at first. In that case just call `s` again
without any parameters. The script is also on
[Gitlab](https://gitlab.com/snippets/1787736)

{% highlight Shell %}
#!/bin/bash

IS_DEVICE=<speaker_sink>
BT_DEVICE=<bluetooth_sink>
BT_DEVICE_MAC=<headset_mac_address>

# turn bluetooth on and connect to bluetooth device
hs() {
  echo "power on" | bluetoothctl
  sleep 1 #Sometimes not yet ready
  echo "connect $BT_DEVICE_MAC" | bluetoothctl
}

# turn bluetooth off
is() {
  pacmd set-default-sink $IS_DEVICE
  pacmd set-default-source $IS_DEVICE.monitor
  echo "power off" | bluetoothctl
}

# switch current clients between internal speaker and bluetooth device
switch() {
  if [[ $(cat ~/.hs_status) == 'off' ]]; then
    echo "on" > ~/.hs_status
    move_to_bluetooth
  else
    echo "off" > ~/.hs_status
    move_to_internal_speaker
  fi
}

# set default device and move current clients to bluetooth device
move_to_bluetooth() {
  echo "Moved to bluetooth."
  pacmd set-default-sink $BT_DEVICE
  pacmd set-default-source $BT_DEVICE.monitor
  for playing in $(pacmd list-sink-inputs | awk '$1 == "index:" {print $2}')
  do
    pacmd move-sink-input $playing $BT_DEVICE
  done
  for recording in $(pacmd list-source-outputs | awk '$1 == "index:" {print $2}')
  do
    pacmd move-source-output $recording $BT_DEVICE.monitor
  done
}

# set default device and move current clients to interal speaker
move_to_internal_speaker() {
  echo "Moved to internal speakers."
  pacmd set-default-sink $IS_DEVICE
  pacmd set-default-source $IS_DEVICE.monitor
  for playing in $(pacmd list-sink-inputs | awk '$1 == "index:" {print $2}')
  do
    pacmd move-sink-input $playing $IS_DEVICE
  done
  for recording in $(pacmd list-source-outputs | awk '$1 == "index:" {print $2}')
  do
    pacmd move-source-output $recording $IS_DEVICE.monitor
  done
}

if [[ $1 ]]; then
  case "$1" in
    hs) hs ;;
    is) is ;;
    switch) switch ;;
    move) switch ;;
  esac
# without any argument
else
  # read current state
  if [[ $(cat ~/.hs_status) == 'on' ]]; then
    # apply current state for filthy clients again
    move_to_bluetooth
  else
    # apply current state for filthy clients again
    move_to_internal_speaker
  fi
fi
{% endhighlight %}

### Button Interface

You should now be able to receive `XF86AudioPrev`, `XF86AudioNext`,
`XF86AudioPlay`, `XF86AudioPause` key presses or whatever your bluetooth
headset is able to send. `xev` helps you with this task.

#### Awesome-wm

With `playerctl` installed you can register the following key objects inside
your `rc.lua`.

{% highlight Lua linenos %}
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

### Acknowledgement

Thanks to [@nagua](https://github.com/nagua/) for pointing me into the direction of `udev`.
