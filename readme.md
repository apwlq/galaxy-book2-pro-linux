# Linux on the Samsung Galaxy Book2 Pro

I am running Ubunutu 22.10 with the Ubuntu-packaged kernel version 5.19 on my [Samsung Galaxy Book2 Pro (NP950XED-KA2SE)](https://www.samsung.com/se/business/computers/galaxy-book/galaxy-book2-pro-15inch-i7-16gb-512gb-np950xed-ka2se/). This repository contains various notes different configurations which I am using.

```sh
$ sudo dmesg
[    0.000000] microcode: microcode updated early to revision 0x421, date = 2022-06-15
[    0.000000] Linux version 5.19.0-28-generic (buildd@lcy02-amd64-027) (x86_64-linux-gnu-gcc-12 (Ubuntu 12.2.0-3ubuntu1) 12.2.0, GNU ld (GNU Binutils for Ubuntu) 2.39) #29-Ubuntu SMP PREEMPT_DYNAMIC Thu Dec 15 09:37:06 UTC 2022 (Ubuntu 5.19.0-28.29-generic 5.19.17)
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-5.19.0-28-generic root=UUID=51750a71-2075-49d1-b42a-895d4b9c3ebb ro quiet splash i915.enable_dpcd_backlight=3 vt.handoff=7
...
[    0.000000] DMI: SAMSUNG ELECTRONICS CO., LTD. 950XED/NP950XED-KA2SE, BIOS P08RGF.054.220817.ZQ 08/17/2022
...

$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 22.10
Release:	22.10
Codename:	kinetic
```

In order to install Linux on the laptop you will need to adjust the BIOS setting for Secure Boot to either
- With `Secure Boot Control` set to `On`, set `Secure Boot Certificate keyset` to `Secure Boot Supported OS` if you wish to run Linux but only with kernels signed with the certificate to support Secure Boot (including those from Ubuntu's default kernel packages).
- or just set `Secure Boot Control` to `Off` if you want to run unsigned kernels (such as if you wish to compile the kernel yourself).

## Display Backlight

To add support for controlling the OLED display backlight you can add the following boot parameter: `i915.enable_dpcd_backlight=3`

For example with Grub you would modify the file `/etc/default/grub` and add it to the `GRUB_CMDLINE_LINUX_DEFAULT` value, run `sudo update-grub`, and then reboot.

I have created an issue to try and drive an upstream fix here: [freedesktop.org: Wrong backlight control type on Samsung Galaxy Book 2 Pro](https://gitlab.freedesktop.org/drm/intel/-/issues/7972).

## Display Out with Thunderbolt 3/4 Dock

- Make sure to use a high quality cable which specifically supports Thunderbolt 3 or 4 (depending on your dock).
- If your dock relies on the graphics hardware of the device (for example DisplayPort Alt Mode) as opposed to a software-based solution (like DisplayLink) then what I have found is:
  - The USB-C port closer to the back (by the full-size HDMI port) works with HDMI to the monitor
  - The USB-C port closer to the front (by the power/charging light) works with DisplayPort to the monitor
- Maybe try setting the refresh rate a bit lower than the maximum otherwise I have noticed some instability / intermittent failures.
- Power Delivery through the dock seems to work very well if you have plugged in before booting, but if you plug the dock into the front (near power light) USB-C port then sometimes Power Delivery does not work. If you plug into the back port first, then unplug and plug into the front, it sometimes "kicks on" a bit more reliably.

If using a lower quality USB-C cable with a dock I found that it can work with HDMI out to the monitor if you disable MST using the following kernel options:

- `i915.enable_dp_mst=0`
- `i915.enable_psr2_sel_fetch=1`

(For example by updating `/etc/default/grub`, running `update-grub`, and rebooting)

This works pretty well if you boot the kernel with the cable already connected, but it can be a little tricky to force the display to come on if you connect afterwards (you basically have to enable/disable output multiple times in a certain sequence before it will come on).

## Keyboard Backlight

TODO keyboard and OS setting are not working

- "Always on" which is cool, but maybe we would want to turn it off sometimes?
- No LED device present for it in `/sys/class/leds/` ?
- The Fn key (Fn+F9) is not recognized

```sh
[ 7441.642453] atkbd serio0: Unknown key pressed (translated set 2, code 0xac on isa0060/serio0).
[ 7441.642465] atkbd serio0: Use 'setkeycodes e02c <keycode>' to make it known.
```

TODO: I have a working thought that this (along with some other settings) should be controlled via a new `samsung-wmi` driver, similar to how it works for ASuS, MSI, etc, but not Samsung as of yet. [Here](https://github.com/gh2o/samsung-wmi) is some inspiration where it seems like someone did something like this before?

I have created some working files in the `wmi` folder of this repository. It seems like the WMI GUID has been updated to `C16C47BA-50E3-444A-AF3A-B1C348380002` (from `01`) and no idea if the "magic" that this example seems to be doing still holds up. Next step to take the MOF files into Windows and take a peek with [wmimofck.exe](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/using-wmimofck-exe) to see if I get better results compared to [bmfdec](https://github.com/pali/bmfdec) (which generated some warnings).

Samsung also seems to be focusing on WMI more with recent Windows devices? For example with the [Samsung Galaxy Book BIOS WMI Guide](https://pcmanagement.biz.samsung.com/support-resources/bios-wmi-guide/) it seems like they have made an investment to use WMI in general for this device?

## Fingerprint Reader

There is currently no [libfprint](https://fprint.freedesktop.org/) driver for the fingerprint reader sensor in this device. I have reverse-engineered and started a PoC in Python that is a start for how a driver could be built for the ELAN / EgisTec `1C7A:0582` device on this laptop. See [fingerprint/readme.md](./fingerprint/readme.md) for more details.

## Sound

Audio works over bluetooth, with the 3.5mm audio jack, or with the USB-A or C ports, but not out-of-the-box with the speakers.

There is quite a bit of discourse online about this issue, IMO the best collection of information is in this Github Issue posted on the SOF project here: https://github.com/thesofproject/linux/issues/4055 (including a reference to the Manjaro thread here: https://forum.manjaro.org/t/howto-set-up-the-audio-card-in-samsung-galaxy-book/37090)

There I have even posted a pastebin of my file [necessary-verbs.txt](sound/necessary-verbs.txt).

If you like, you can run the script [necessary-verbs.sh](sound/necessary-verbs.sh) to "turn on" the speakers but note that you will need to run this periodically and/or create some kind of service that runs it in the background on certain events or a certain schedule (a bit like is shown in the Manjaro thread linked above).

Some of the config files and other logs from my capturing of this list (essentially: running Windows in a QEMU container, mapping the audio devices to the QEMU container, installing the audio drivers within the virtual Windows environment, playing sound from within the QEMU container, and capturing the output into a log file) you can find in the [sound](./sound/) folder of this repository.

TODO how to continue narrowing down the necessary verb list and then package it for an upstream patch?

## Battery

I typically get around 5-7 hours of battery life in Linux. Here are some tips:

- Turn off Bluetooth if you do not need to regularly use it.
- Install `powertop` and then run both `powertop --auto-tune` and `powertop --calibrate` (note that calibration does take some time and does funny stuff with the screen brightness!).
- Either in the Windows Samsung app or in the BIOS turn on the setting which stops charging at around 85%
- Charge when battery gets near 20%, and remove the cable once it stops at 85%
