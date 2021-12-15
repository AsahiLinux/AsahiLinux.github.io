+++
date = "2021-12-14T12:00:00+09:00"
draft = false
title = "Progress Report: October-November 2021"
slug = "progress-report-oct-nov-2021"
author = "marcan"
+++

Whoops, things got so busy we ended up skipping a month! A lot has been going on in kernel land over the past couple of months, so we hope you enjoy this combined October-November progress report!

## M1 Pro/Max joins the family

At the end of October, Apple launched the next generation of Apple Silicon: M1 Pro and M1 Max. We got right to work on supporting these new machines, and after just a [few days of work](https://www.youtube.com/playlist?list=PL68XxS4_ek4bo7umPx1AuhGHAUh-lVUGs) we were able to bring up Asahi Linux on them up to feature parity with the M1 machines! Going forward, we'll be supporting these machines as well as the previous generation.

{{< captioned caption="glxgears has no right to run at >60FPS with software rendering, but it does." >}}
    {{< tweet 1458473546225577987 >}}
{{< /captioned >}}

It's interesting to take a peek at what Apple have done with the new chips. The original M1 (codename "Tonga", SoC name T8103) would have more properly been called the A14X following Apple's existing naming convention; it is, in fact, the tablet version of the A14 iPhone SoC (T8101). In order to build a Mac-worthy chip, Apple added some critical features (like Thunderbolt support), but otherwise left the old iPhone-centric architecture largely intact.

While this worked well for the machine classes the M1 shipped in, the A14 class was running into limitations scaling up to larger SoCs. This isn't due to a lack of foresight by Apple, but rather because they strive to keep backwards compatibility with older SoCs as much as possible. This means certain design elements of the A14 date back to much older iPhone chips, from an era where 16GB of RAM or more than 16 cores seemed completely ludicrous for mobile devices.

With the M1 Pro/Max (codename "Jade", SoC names T6000/T6001), Apple iterated on the M1 architecture in order to prepare for scaling it up in future chips. While keeping the core components the same (M1 Pro/Max have many of the same components as M1, including the same CPU cores), they broke backwards compatibility in two critical areas where the M1 ran into limitations: the interrupt controller, and the physical address space size (which required changing the IOMMUs).

The interrupt controller is now Apple Interrupt Controller 2, breaking with many years of backwards compatibility with iPhone SoCs that used AIC1. AIC1 was a fairly simple controller with support for up to 32 CPU cores (including coprocessors, so in practice fewer "normal" cores), and a legacy IPI feature that was no longer used on the M1. AIC2 got rid of some of this legacy stuff and removed CPU affinity support entirely. Instead, there is now an automagic heuristic that picks which CPU to deliver an interrupt to, which can be influenced by configuration decisions from that CPU itself as well as its state (idle, active, interrupts enabled, etc). This should effectively let AIC2 scale infinitely to as many cores as necessary, as there is no longer any part of the interrupt controller itself that has to select individual CPU cores.

While working on AIC2 we discovered an interesting feature... while macOS only uses one set of IRQ control registers, there was indeed a full second set, unused and apparently unconnected to any hardware. Poking around, we found that it was indeed a fully working second half of the interrupt controller, and that interrupts delivered from it popped up with a magic "1" in a field of the event number, which had always been "0" previously. Yes, this is the much-rumored multi-die support. The M1 Max SoC has, by all appearances, been designed to support products with two of them in a multi-die module. While no such products exist yet, we're introducing multi-die support to our AIC2 driver ahead of time. If we get lucky and there are no critical bugs, that should mean that Linux just works on those new 2-die machines, once they are released!

The physical address space size of the CPU cores increased from 36 bits (64 GiB) to 42 bits (4 TiB). The RAM region moved from the top 32 GiB of address space to the top 3 TiB. This means that while the old M1 chips were architecturally limited to a maximum of 32 GiB of RAM, the M1 Pro/Max revision can support up to at least 1 TiB or 3 TiB (depending on whether they want to keep it power-of-two aligned or not). Note that this is not how much memory the chips support in actuality, just how high the underlying architecture can scale without incompatible changes.

Supporting this increased physical address space required changes to the IOMMUs in the system, including the DARTs (for most peripherals) and the SART (for the NVMe controller). Amusingly, while implementing support for this in Linux, we ran into a bug in Linux's ARM SMMU support that had been there ever since 52-bit address support was introduced. This was breaking systems with more than 256 TiB of RAM - I wonder why nobody noticed? Either way, Linux now correctly supports standard ARM systems with up to 4 PiB of RAM ;-).

The rest of the M1 Pro/Max bring-up was smooth sailing; with a number of tweaks to our bootloader [m1n1](https://github.com/asahilinux/m1n1) and new device trees, no other changes to Linux itself were required. This is a bet we placed early on when beginning M1 development: that Apple would not break compatibility unnecessarily, and that we could keep finicky per-SoC details handled in m1n1 instead of Linux. So far this has been the case, and we don't anticipate the DARTs or AIC changing again for quite a while. This should allow older kernels to boot on newer, future hardware and SoCs (at least enough to e.g. run a distro installer and later update your kernel to one with full support), which is almost unheard of in embedded ARM systems of this style.

New features in the new M1 Pro/Max MacBooks include the HDMI port, the more complex speaker configuration, and the SD card reader. The HDMI port is not supported yet, as external displays aren't supported at all on any machine (except the HDMI port on the Mac Mini, of course), but that will come along with general external display support in the future. The SD card reader required a few quirks to make work, but these have already been submitted upstream and it works great! Finally, audio support will be added as part of the ongoing audio driver work.

I know what you're all thinking now... what about the notch (and the rounded top corners)? There is currently no explicit support for the notch, though as you can see in the tweet above, it's not hard to configure the KDE Plasma panels to match the macOS layout and largely make the notch work out! That said, our plan for this is to initially exclude the notch from the screen resolutions presented to userspace in the proper display driver, so that existing notch-naive desktop environments can work without any changes. In the future, as we work out how this should be supported in the desktop, we will enable opt-in notchful resolutions that notch-aware compositors can select. Fun fact: the notch and corner edge pixels are properly anti-aliased in *hardware*. Apple really don't *cut corners* when it comes to making their hardware friendly to software developers :-).

Finally, it's interesting to note that the M1 Pro really is just a cut-down version of the M1 Max - not physically (that's not a thing, you can't chop chips in half), but certainly design-wise. All the interrupt numbers and hardware addresses are the same. This means that we can effectively treat them as the same SoC. Our M1 Pro device tree just includes the M1 Max one, and then disables/removes the missing bits.

## A wild linux-asahi appeared!

Over the past year, we've seen lots of development happening in separate kernel branches, but there wasn't any "official" kernel branch collecting work before it is upstreamed. Now that many drivers are landing upstream and platform device trees are settling down, it's time to start collecting our ongoing work into a common branch.

Say hi to the [linux-asahi](https://github.com/AsahiLinux/linux/tree/asahi) kernel branch! This tree is a bleeding edge kernel based on recent upstream development, with changes and new drivers that are either already on their way upstream, or are in a usable enough state that we would like people to test them. Going forward, we expect most users to be using this kernel if they want to test the latest things we've been working on.

Note that this branch is currently based on linux-next, and therefore should not be considered a stable kernel. As things move on, it will probably shift to being based on RCs and even stable kernel versions in the future. We're also doing frequent rebasing, so although we encourage developers to base off of it if they're working on something new and need features not yet available in mainline, please make sure you're comfortable with git rebasing and juggling commits before you do so!

In addition, we also have branches under the `asahi-soc/` path, such as [asahi-soc/dt](https://github.com/AsahiLinux/linux/tree/asahi-soc/dt). These trees represent the Apple platform work that we are directly responsible for as Apple ARM platform maintainers for Linux; chiefly, this means Device Tree changes and certain drivers (notably, PMGR, and soon the RTKit layer). There is also an [asahi-soc/next](https://github.com/AsahiLinux/linux/tree/asahi-soc/next) branch that contains the latest changes merged into linux-next (sporadically updated). I must say, directly sending a signed pull request to upstream Linux maintainers still feels a little bit weird!

Note: please don't submit GitHub PRs directly against any of these branches. Due to the frequently rebased nature, this just doesn't work properly, and the development model tends to be different for the kernel (e.g. lots of rewriting history before submitting a patch set upstream, which has implications for authorship). We instead encourage you to talk to us on [#asahi-dev](https://asahilinux.org/community/) if you would like to contribute fixes or new development to the kernel.

## Drivers, drivers, drivers!

The past two months have been full of kernel driver activity! A number of drivers were merged for 5.16, and even more are headed for 5.17. Let's take a look at the status of hardware support in convenient table form:

<style>
table.features {
  padding: 10px;
  font-weight: bold;
}

div.legend span {
  padding: 0 5px;
  font-weight: 500;
}

table.features td, table.features th {
  padding: 4px;
  text-align: center;
  border: 1px solid #ccc;
  vertical-align: middle;
}

tr.sub th:nth-child(1n+2) { font-size: 70%; }
td.mrg, span.mrg { background-color: #080; color: #fff; }
td.nxt, span.nxt { background-color: #088; color: #fff; }
td.rev, span.rev { background-color: #06c; color: #fff; }
td.alx, span.alx { background-color: #00c; color: #fff; }
td.wip, span.wip { background-color: #c0c; color: #fff; font-weight: 200; }
td.ehh, span.ehh { background-color: #c22; color: #eee; font-weight: 200; }
td.nul, span.nul { background-color: #666; color: #ccc; font-weight: 200; }
</style>

Legend:

<div class="legend">

* <span class="mrg">Merged and released in upstream Linux</span>
* <span class="nxt">Merged for an upcoming Linux release</span>
* <span class="rev">In review, likely to be merged by this release</span>
* <span class="alx">Work in progress, available for testing in the linux-asahi branch</span>
* <span class="wip">Work in progress, not yet ready for broad testing</span>
* <span class="ehh">Patches available, but requiring a major rewrite</span>
* <span class="nul">Not yet supported</span>

</div>

<table class="features">
<thead>
<tr>
  <th></th>
  <th colspan="4">M1 (T8103)</th>
  <th>M1 Pro (T6000)</th>
  <th>M1 Max (T6001)</th>
</tr>
<tr>
  <th></th>
  <th>Mac Mini</th>
  <th>13" MBA</th>
  <th>13" MBP</th>
  <th>iMac</th>
  <th>14"/16" MBP</th>
  <th>14"/16" MBP</th>
</tr>
<tr class="sub">
  <th>Feature</th>
  <th>j274</th>
  <th>j313</th>
  <th>j293</th>
  <th>j456 / j457</th>
  <th>j314s / j316s</th>
  <th>j314c / j316c</th>
</tr>
</thead>
<tbody>
<tr><th>Core bring-up</th>
  <td class="mrg" colspan="4">5.13</td>
  <td class="rev" colspan="2">5.17</td>
</tr>
<tr><th>Device Tree</th>
  <td class="mrg">5.13</td>
  <td class="nxt" colspan="3">5.17</td>
  <td class="alx" colspan="2">5.17</td>
</tr>
<tr><th>CPU PMU</th>
  <td class="rev" colspan="6">5.18</td>
</tr>
<tr><th>CPU Frequency</th>
  <td class="ehh" colspan="6">Rewrite soon</td>
</tr>
<tr><th>CPU Deep Sleep</th>
  <td class="nul" colspan="6">Blocked on PSCI discussion</td>
</tr>
<tr><th>System Sleep</th>
  <td class="nul" colspan="6">Blocked on PSCI discussion</td>
</tr>
<tr><th>UART</th>
  <td class="mrg" colspan="6">5.13</td>
</tr>
<tr><th>Watchdog (reboot)</th>
  <td class="rev" colspan="6">5.17</td>
</tr>
<tr><th>PCIe</th>
  <td class="nxt" colspan="6">5.16</td>
</tr>
<tr><th>I²C</th>
  <td class="nxt" colspan="6">5.16</td>
</tr>
<tr><th>GPIO</th>
  <td class="nxt" colspan="6">5.16</td>
</tr>
<tr><th>USB-PD</th>
  <td class="nxt" colspan="6">5.16</td>
</tr>
<tr><th>MagSafe</th>
  <td colspan="4"></td>
  <td class="nxt" colspan="2">5.16</td>
</tr>
<tr><th>Device PM</th>
  <td class="nxt" colspan="6">5.17</td>
</tr>
<tr><th>NVMe</th>
  <td class="alx" colspan="6">Works well, needs cleanup</td>
</tr>
<tr><th>SPI</th>
  <td class="rev" colspan="6">5.17</td>
</tr>
<tr><th>SPI NOR Flash</th>
  <td class="rev" colspan="6">5.17</td>
</tr>
<tr><th>SPI HID transport</th>
  <td></td>
  <td class="alx" colspan="2">New!</td>
  <td></td>
  <td class="alx" colspan="2">New!</td>
</tr>
<tr><th>Internal Keyboard</th>
  <td></td>
  <td class="alx" colspan="2">(↑)</td>
  <td></td>
  <td class="alx" colspan="2">(↑)</td>
</tr>
<tr><th>Internal Touchpad</th>
  <td></td>
  <td class="alx" colspan="2">New!</td>
  <td></td>
  <td class="alx" colspan="2">New!</td>
</tr>
<tr><th>Touch Bar (touch)</th>
  <td colspan="2"></td>
  <td class="nul">HID?</td>
  <td colspan="3"></td>
</tr>
<tr><th>Touch Bar (display)</th>
  <td colspan="2"></td>
  <td class="nul"></td>
  <td colspan="3"></td>
</tr>
<tr><th>Primary display (SimpleFB)</th>
  <td class="mrg" colspan="6">5.13</td>
</tr>
<tr><th>Primary display (SimpleDRM)</th>
  <td class="nxt" colspan="6">5.17</td>
</tr>
<tr><th>Primary display (DCP)</th>
  <td class="wip" colspan="6">Works, needs work</td>
</tr>
<tr><th>Backlight</th>
  <td></td>
  <td class="nul" colspan="5"></td>
</tr>
<tr><th>DisplayPort</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>HDMI</th>
  <td class="mrg">(Primary)</td>
  <td colspan="3"></td>
  <td class="nul" colspan="2">(↑) Internally DisplayPort</td>
</tr>
<tr><th>Thunderbolt</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>USB2 (TB ports)</th>
  <td class="rev" colspan="6">5.17 (quirks)</td>
</tr>
<tr><th>USB3 (TB ports)</th>
  <td class="nul" colspan="6">Understood; needs PHY driver</td>
</tr>
<tr><th>USB2/3 (C ports)</th>
  <td colspan="3"></td>
  <td class="nul">FW issue</td>
  <td colspan="2"></td>
</tr>
<tr><th>USB2/3 (A ports)</th>
  <td class="nxt">5.16</td>
  <td colspan="5"></td>
</tr>
<tr><th>Ethernet (1G)</th>
  <td class="nxt">5.16</td>
  <td colspan="2"></td>
  <td class="nxt">5.16</td>
  <td colspan="2"></td>
</tr>
<tr><th>Ethernet (10G)</th>
  <td class="nxt">5.17</td>
  <td colspan="5"></td>
</tr>
<tr><th>SD Card Reader</th>
  <td colspan="4"></td>
  <td class="rev" colspan="2">5.17</td>
</tr>
<tr><th>Wi-Fi</th>
  <td class="ehh" colspan="6">Rewrite soon + firmware handling</td>
</tr>
<tr><th>Bluetooth</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>SMC (Batt/Fans/etc)</th>
  <td class="nul" colspan="6">Soon; m1n1 experiment available</td>
</tr>
<tr><th>Audio Playback</th>
  <td colspan="6" class="alx">New!</td>
</tr>
<tr><th>Audio Capture</th>
  <td colspan="6" class="nul"></td>
</tr>
<tr><th>Speakers</th>
  <td class="alx">New!</td>
  <td colspan="5" class="nul">Needs amp driver or config</td>
</tr>
<tr><th>Microphones</th>
  <td colspan="6" class="nul"></td>
</tr>
<tr><th>Headphones Jack</th>
  <td colspan="4" class="alx">New!</td>
  <td colspan="2" class="nul">Needs codec driver</td>
</tr>
<tr><th>SPMI</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>RTC</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>SEP</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>Touch ID (Internal)</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>Touch ID (External)</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>Video Decode</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>Video Encode</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>JPEG</th>
  <td class="nul" colspan="6">Worth it?</td>
</tr>
<tr><th>Scaler</th>
  <td class="nul" colspan="6">Useful?</td>
</tr>
<tr><th>ProRes</th>
  <td colspan="4"></td>
  <td class="nul" colspan="2"></td>
</tr>
<tr><th>GPU</th>
  <td class="nul" colspan="6">Userspace in good shape, kernel work starting soon</td>
</tr>
<tr><th>Neural Engine</th>
  <td class="nul" colspan="6"></td>
</tr>
<tr><th>Camera (ISP)</th>
  <td></td>
  <td class="nul" colspan="5"></td>
</tr>
</tbody>
</table>

Of the drivers mentioned in the [previous progress report](/2021/10/progress-report-september-2021/), Pinctrl, I²C, ASC Mailbox, and Device Power Management have now been merged. On top of that, we have the new M1 Pro/Max AIC2, CPU PMU, and SPI controller drivers in review, as well as fixes to make a bunch of devices work properly: Aquantia NIC MAC address handling for 10G Mac Mini, SD card reader quirks for the new MacBooks, fixes to make the new SimpleDRM driver work like SimpleFB did, quirks to make USB2 hotplug work, and more.

CPU frequency scaling was submitted, but we ended up deciding to try to approach the driver in a different way after the discussion, so that is due for a rewrite soon. That said, m1n1 currently initializes all cores to a decent performance level (2GHz) by default, so missing the 5.17 merge would not be a big deal; we're prioritizing drivers which enable critical pieces of hardware (such as the internal keyboard) for upstreaming first.

[Martin Povišer](https://github.com/povik) has been hard at work on the audio hardware, and his Linux ASoC driver can now drive the internal speaker on the Mac Mini and the headphones jack on all the M1 Macs! Speaker support and jack support on the Max/Pro machines just depends on supporting the various speaker amp / codec chips. On top of this, he will be looking into how to correctly drive the multiple-speaker setups in some of the laptops, as they need to have crossover filters implemented in software; this will involve figuring out how to express this requirement in userspace so ALSA, PulseAudio, PipeWire and friends can treat the speaker arrangement as a stereo pair as far as users are concerned.

Finally, [Janne Grunau](https://twitter.com/jaguar_9) has just written a brand new Apple HID over SPI transport driver to support the keyboard and touchpad on the MacBooks. It turned out that Linux already had a total of *3* drivers to process Apple-format touchpad events! One of them was written for Intel systems using SPI, but without the understanding that the underlying protocol is HID; one was written to work on raw USB devices (touchpads in USB mode); and one is a proper HID layer driver intended to work with external Magic Mouse and Magic Touchpad devices via USB or Bluetooth. We'd been testing with patches to the existing applespi keyboard/touchpad driver, but it is clear that this is not the right way to go, as the driver duplicates a lot of existing kernel functionality incompletely. Instead, by writing a new (much simpler) HID transport driver, we can work with Linux's standard HID keyboard/mouse support and the existing Magic Touchpad HID driver (possibly with some protocol additions). In the future, we hope this will replace applespi on Intel systems too, and the entire saga ends up with the kernel losing a couple thousand lines of redundant code.

## Device trees for all

With drivers coming together for devices that are configured differently across the different machines, the time had come to write proper device trees covering each device specifically. Janne Grunau submitted a series to add all the original M1 Mac device trees, and we've merged it through our asahi-soc tree; it will be upstream in 5.17. This covers the M1 Mac Mini, MacBook Air, MacBook Pro, and the two iMac variants (2 USB ports and 4). In addition, device trees for the M1 Pro and Max machines are in our linux-asahi branch, and will be upstreamed as soon as the new bindings and drivers for these machines get merged (hopefully for 5.17 too).

## U-Boot

For end-user installs of Asahi Linux, we will be using U-Boot as a boot stage to provide EFI services and basic I/O, so that distributions can interact with a standard-looking UEFI environment. Mark Kettenis has been hard at work getting M1 support into U-Boot, and the initial support patches have been merged and will be part of U-Boot 2022.01!

## Next steps

Our next area of focus will be Wi-Fi. There has been a Wi-Fi support patch floating around since earlier this year, but unfortunately the way it handles firmware is completely backwards, as it requires users to dig up the specific firmware required for their machine (as well as manually set the MAC address) instead of auto-selecting the correct one. This needs a complete rewrite and coordination with firmware copying tools. We have a pretty good idea as to how this should be designed, so we will be integrating the firmware copying and renaming into our bootloader installer, and then we'll write a new patch that requests the correct firmware for the platform it runs on. This allows the driver to work without users having to do anything, and makes the Linux installation portable between different machines (at least as far as the root filesystem). This will also work for T2 Macs with some extra work to fetch some information from ACPI. Stay tuned for updates on this in the coming week or two!

We're also going to be looking at SMC, which is responsible for things like battery management, fan and temperature control, etc. Macs have had SMCs since time immemorial, and the ones in M1 macs just carry over the same concept on top of the RTKit ASC coprocessor
framework that we already know well. The core interface is a simple key/value store, so most of the work here will be adding support for all the useful keys available, as sub-drivers to handle different aspects of the hardware.

SPMI and RTC support are also nice and simple drivers that we'll be tackling soon; although we all run NTP time synchronization, nobody likes their computer booting up in 1970, so that's another important checkbox to tick. Thankfully, this should be quite easy. In fact, thanks to Mark Kettenis' efforts, OpenBSD [already](https://man.openbsd.org/aplpmu.4) supports this hardware, so we already know how it works!

While testing kernels on multiple machines, I found that the extra 2 USB C ports on the iMac did not work. After a long debugging session, it turned out that the USB controller in these machines requires firmware to be uploaded to it during startup. This will depend on our upcoming firmware copying infrastructure, so it is on the back burner for now, but the kernel changes required to make it work are not complicated.

And, of course, once these things are taken care of, it'll be time to tackle the GPU kernel driver! 2022 is definitely going to be an exciting year for Asahi Linux.
