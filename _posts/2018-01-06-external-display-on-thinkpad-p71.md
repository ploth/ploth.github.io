---
title: External Display on Thinkpad P71
tags: nvidia nouveau external graphics bumblebee bbswitch thinkpad
---

## External display on Thinkpad P71

The following setup might be a little bit different from what you see in the
wild but I'm quite happy with it. As you might already noticed it is a pain in
the ass when it comes to Nvidia Optimus on Linux. It doesn't make it any better
that you need to load up the `nvidia-driver` or respectively
`xf86-video-nouveau` to make use of a plugged-in display. `xf86-video-intel`
can't address the external monitor ports. In the following I would like to
present you my setup.  

Every time before I start the X server, I think about whether I need the Nvidia.
Accordingly I fire up my nvidia script. It toggles the power state of the
discrete graphics, symlinks to the correct xorg configuration file, un-/loads
corresponding kernel modules and switches the OpenGl and OpenCl implementation.
Then I'm ready to start my X server.

Most of the time I don't need the discrete Nvidia GPU when on battery so it's
deactivated. Bonus: Having 7 hours of battery life. To toggle the power state
I'm using [bbswitch](https://github.com/Bumblebee-Project/bbswitch). Make sure
you add `bbswitch` to `/etc/conf.d/modules` so it loads up when you start your
system.

I've set up two xorg configurations. You may have to adapt them to your system.

#### /etc/X11/xorg.conf.intel
{% highlight Shell linenos %}
Section "Device"
	Identifier "Intel Graphics"
	Driver "intel"
EndSection
{% endhighlight %}

#### /etc/X11/xorg.conf.nvidia
{% highlight Shell linenos %}
Section "Device"
	Identifier "NVIDIA Graphics"
	Driver "nvidia"
	BusID "PCI:1:0:0"
	Option "AllowEmptyInitialConfiguration"
	Option "Coolbits" "28"
	Option "NoLogo" "1"
EndSection

Section "Module"
	Load "modesetting"
EndSection
{% endhighlight %}

Before I start X, all I have to do is to write `nvidia on` respectively `nvidia off`. 

#### ~/bin/nvidia
{% highlight Shell linenos %}
#!/bin/bash

[[ $UID != 0 ]] && exec sudo "$0" "$@"

on() {
        echo ON > /proc/acpi/bbswitch
        ln -svfn /etc/X11/xorg.conf.nvidia /etc/X11/xorg.conf.d/20-graphics.conf
        modprobe nvidia
        modprobe nvidia_modeset
        modprobe nvidia_drm
        eselect opengl set nvidia
        eselect opencl set nvidia
}

off() {
        rmmod nvidia_drm
        rmmod nvidia_modeset
        rmmod nvidia
        eselect opengl set xorg-x11
        eselect opencl set intel
        ln -svfn /etc/X11/xorg.conf.intel /etc/X11/xorg.conf.d/20-graphics.conf
        echo OFF > /proc/acpi/bbswitch
}

case "$1" in
        on) on ;;
        off) off ;;
        *) echo "Usage: $0 on|off" >&2 ;;
esac
{% endhighlight %}
