+++
date = "2022-07-18T10:00:00+02:00"
draft = false
title = "M2 is here! July 2022 Release & Progress Report"
slug = "july-2022-release"
author = "marcan"
+++

Welcome to another long overdue progress report! As usual, things have been busier than expected… and we have some big news! We’ve just released a new Asahi Linux update with Mac Studio, Bluetooth, and *M2 support*!

If you're new to Asahi Linux, check out our [previous release announcement](https://asahilinux.org/2022/03/asahi-linux-alpha-release/) for installation instructions and general information.

## Mac Studio joins the family

When the Mac Studio was announced, we set to work making the new M1 Ultra work with Asahi Linux. This wasn’t hard, but it did need some changes to our bootloader and device trees in order to handle the idea of one SoC with multiple dies. This eventually got rolled in with other changes, so we ended up waiting a bit longer than we expected to release it, but it’s finally here!

You can expect most of the hardware to work as you’d expect (on par with the Mac Mini), except for the front USB ports on the M1 Max model and the Type A ports on all models (these are blocked on the special firmware upload support also needed for the 4-port version of the M1 iMac - it’s on the list).

## A wild Bluetooth appeared!

Bluetooth had been on the back burner for a while now, since Apple switched to a new bespoke PCIe interface that apparently no other vendor uses. But all that changed when [R](https://twitter.com/rqou_) picked up the challenge of reverse engineering it! Thankfully, Bluetooth itself is quite simple, since the host to controller interface is largely standardized. Apple made a variant that runs over PCIe, but the higher layers are the same as any other Bluetooth controller. After R put together a userspace proof of concept driver, Sven picked up the work and started writing a proper kernel driver. As of a few days ago, Bluetooth started working!

We’ve decided to take the plunge and ship this driver straight to alpha users, and it is now available in our latest kernel and support packages. Thankfully, while the PCIe transport is new, the HCI interface that runs on top is standard, so once the core initialization and data transfer parts of the driver started working, most Bluetooth features did too. The driver does not need to concern itself with any of those details, it just shuffles data to/from the device. There is, however, one caveat: WiFi/Bluetooth coexistence isn't properly configured yet, so you will have poor Bluetooth performance if you are connected to a 2.4GHz WiFi network. We recommend turning off WiFi or using a 5GHz network if you want to use Bluetooth at this time; we will be adding coexistence support in the coming weeks.

In addition to the driver, Bluetooth support is the first major test of the “seamless upgrade” paradigm we’ve been aiming for. While Asahi Linux is still an experimental project, we knew that we didn’t want to force users to go through arduous manual steps to get new things to work. Getting Bluetooth enabled involved a lot of small changes to different parts of the stack:

* m1n1 changes to forward the Bluetooth address and calibration information from the Apple Device Tree to the Linux Device Tree.
* Device Tree changes to add the new device and specify the hardware board type for each specific machine
* Installer changes to extract the Bluetooth firmware and place it in the correct location for Linux to find
* A completely new Linux driver
* New default package additions for Plasma desktop users, plus `bluetooth.service` needs to be enabled.

We knew this would happen from the get go, so we designed our reference distro to support doing all of that, automatically:
* m1n1 is split into two stages, and the second stage (which has most of the logic) can be upgraded as a normal package.
* The device trees are bundled with the Linux kernel and upgraded into m1n1 when the kernel is, which always keeps them in sync.
* Since we knew the existing installer was missing some firmware, we made it create a “fallback” archive of all the raw firmware (before processing) so that we could re-process it from Linux. We’ve now added an `asahi-fwextract` package that contains the firmware processing components of the installer, and automatically upgrades your firmware bundle when installed, using the fallback archive as input.
* We shipped empty meta packages with our default installs, which allow us to add new packages as dependencies for existing users. We’ve used them to install `asahi-fwextract` for everyone, and the miscellaneous BlueZ/Bluedevil/etc packages needed for a good Bluetooth experience for desktop users. It also enables `bluetooth.service` on upgrade.

The end result is that existing reference distro users can simply upgrade their packages, reboot, and Bluetooth should just work without any further configuration or changes!

## M2 is here!
Ever since Asahi Linux started, one of the most common questions we get asked is “what about the M2?” Indeed, the M1 was only the first step, and Apple aren’t going to stop releasing new chips and machines any time soon. How does this affect the Asahi Linux project? We’ve long held that we don’t expect porting to newer chips to be nearly as much of a challenge as the first time, and that many of the drivers would work unmodified… and the M2 is the first true test of this theory. How did we do?

After just a [12-hour bring-up marathon](https://www.youtube.com/watch?v=SidIJkC5YN0), Linux was booting on the M2 with USB, NVMe, battery stats/control, CPUfreq, WiFi, and more! With a few more days of work, we were able to get the keyboard/trackpad working, bringing it to feature parity with existing systems. After some more integration work, we are proud to announce *experimental support for M2 machines in the Asahi Linux installer*!

If you decide to try this on your M2 machine, keep in mind these caveats:
* This is even more experimental than M1 support, so expect bugs. To get the option to install on M2, you need to enable expert mode in the Asahi Linux installer.
* The keyboard won’t work in U-Boot/GRUB. That driver has not been written yet, and we’ve yet to figure out how we want to do the hand-off between U-Boot and Linux. You can use an external USB keyboard if you need to poke around the bootloader shells.
* Only the M2 MacBook Pro 13” is tested. We’ve added completely untested M2 MacBook Air support (because we can), but none of us have one yet! If you do, only try it if you’re feeling very adventurous (and don’t blame us if things go wrong).
* The firmware/stub used for Linux is based on a “special release” macOS 12.4 version that Apple released just for these machines. We are not committing to long-term support for this version just yet, so you *may* have to go through macOS and the installer to upgrade your boot components (likely to 13.0) in order to get future features such as the GPU and external display output to work (i.e. no “seamless upgrade” with just pacman, but you also won’t have to do a full reinstall). We will decide how to proceed in the future, and add the necessary upgrade mode to the installer when the time comes, if necessary. It is possible we will support 12.4 after all, but no promises.

## Trackpad tricks
Let’s take a look at what happened with the trackpad, since it’s an interesting story. Apple MacBooks have historically had an interesting design for their keyboard/trackpad components. While on most x86 laptops these are attached internally via (possibly virtual) PS/2 or sometimes USB, Apple started out with USB and then migrated to SPI. SPI is a simpler protocol than USB, which can save power for a low-bandwidth peripheral like this, that always needs to be ready to respond to user input. Even the latest (pre-T2) Intel MacBooks used SPI already, and this design was carried over to the M1 (we’ll skip T2 machines since they are a special case here).

Apple also use quite a different architecture for their actual keyboard/trackpad. On many x86 machines the keyboard is driven by the Embedded Controller, a dedicated chip on the motherboard that also serves to control a number of other system management features. The trackpad is usually stand-alone. Apple machines, in contrast, have the trackpad in charge of controlling the keyboard - the keyboard is, like on x86 machines, a passive switch matrix, but instead of connecting to the motherboard directly, it connects to the trackpad instead. The trackpad itself is also quite unusual, owing to Apple’s Force Touch technology. Most other laptops use dedicated trackpad chips by Synaptics, but Apple instead takes Broadcom’s BCM5976 - a touchscreen controller with an embedded ARM core, the same thing you would find on an iPhone - and pairs it with an STM32 Cortex-M3 microcontroller. The STM32 (which runs RTKit firmware) is in charge of handling the keyboard and Force Touch parts of the design, and also serves as a front-end to the BCM5976 that combines all the data and implements part of the algorithms that make Apple’s trackpads work as well as they do. Together, those chips have quite a bit of firmware smarts!

Then the M2 came around, and… the SPI interface is gone. Well, not quite gone… but the OS doesn’t see it any more. Instead, the M2 now has another little secret, the MTP. Yes… Apple moved the trackpad controller into the main SoC!

Okay, not exactly. The BCM5976 and STM32 are still there, but both firmwares have shrunk dramatically. Those chips are no longer in charge of any fancy multitouch algorithms or logic; their job has been relegated to being mere interface chips, to get the raw keyboard/multitouch/Force Touch data into the M2. The M2 now has a dedicated coprocessor (probably another ARM64 Chinook instance, running RTKit as usual), which is in charge of the bulk of the processing.

Why would Apple do this? There are a few reasons. First, by integrating the firmware into the M2, this gives them much more flexibility in updating it. This firmware can now be part of the per-OS bundle, which means they can introduce new features and make changes without risking backwards compatibility with older versions of macOS. It also means the firmware upgrade procedure is integrated with macOS upgrades automatically, and is much more robust. And of course, it probably lets them use a smaller STM32 variant and save some money.

With the MTP, Apple handled communications with the main OS in an interesting way. While most coprocessors use the primary mailbox/RTKit interface and shared memory for this, it seems Apple wanted to keep the “bytes in, bytes out” model of the SPI interface, so instead they used… DockChannel. What on Earth is DockChannel, you ask? It’s Apple’s own design for, essentially, a simple FIFO. M1 chips already implement a debug DockChannel that can be used as a virtual serial port, accessible with internal Apple debuggers that connect to a specific Type C port on these Macs (we don’t yet know how this protocol works on the USB side, though we’ve [poked around](https://github.com/asfdrwe/asahi-linux-translations/wiki/HW:Debug-USB) a bit). MTP reused the same FIFO module as a simple byte channel between it and the main OS, with a new HID transport protocol running over it.

The good news is that DockChannel itself is very simple, so bringing that up did not take long. The bad news is that Apple over-complicated the new HID transport (as they do), so that new driver ended up being [over 1000 lines of code](https://github.com/AsahiLinux/linux/blob/asahi/drivers/hid/dockchannel-hid/dockchannel-hid.c) and having to handle things such as firmware upload, GPIO proxying, and multiple complex nested data structures! We also haven’t figured out how to reset MTP and bring it back to a fresh startup state yet, so we cannot yet support handing off between the U-Boot driver and Linux, or removing/re-probing the device on Linux.

To make things even more cursed, it turns out that the firmware for the BCM5976 is loaded on start-up from the host. macOS stores it as an XML plist of all things (with features that the Python XML plist parser does not support) wrapped in an asn.1 img4 image, and the driver needs to convert it to a binary serialized structure (in yet another serialization format…) before handing it off to MTP during initialization, as well as patch in the bInterfaceNumber field with the correct interface number. Needless to say, we are not putting an XML parser in the Linux kernel, so instead the Asahi Linux installer now has a [module](https://github.com/AsahiLinux/asahi-installer/blob/main/asahi_firmware/multitouch.py) in charge of converting it to a binary format. There is a little header containing the bInterfaceNumber offset, so Linux can change it before handing it off to MTP. Phew!

## Ventura adventures

After the macOS 13.0 Ventura beta was released, we received reports that it broke Asahi Linux installs. We’ve long since maintained that macOS upgrades should not break Asahi, since most firmware is associated with an OS (so a macOS upgrade will not affect Asahi). So, what happened?

While most firmware is OS-tied, there is a small subset of System Firmware that is shared by all OSes. This includes the NVMe firmware (for obvious reasons), but also the SMC firmware and some Thunderbolt-related stuff. To maintain compatibility with older versions of macOS, however, Apple essentially pledges to never break backwards compatibility with older firmware, so we should be safe.

And we would’ve been… were it not for bugs! Indeed, m1n1’s NVMe driver had no issues with the updated firmware, and worked right out of the box. U-Boot’s, however, was not so lucky. That version had taken a Clever Shortcut™, and it turned out that the new firmware wasn’t quite happy with our little trick. Meanwhile, the Linux SMC driver had an outright one-line bug that went unnoticed with the old firmware, but was causing the new SMC firmware to crash.

We have now pushed updated linux-asahi, u-boot, and m1n1 packages, so if you want to try out the new betas, make sure you do a pacman upgrade first! If you’ve already upgraded to the beta and got stuck with an unbootable Asahi system, follow the steps [here](https://github.com/AsahiLinux/asahi-installer/issues/100#issuecomment-1162376171) to recover.

Keep in mind that these issues were caused by bugs or oversights in our code, not by Apple breaking compatibility, and we expect these to be less and less common over time. In general, it should always be safe to update a Mac that has Asahi installed, just don’t let the bugs bite :-)

## DCP, please!

The Mac Mini (and now Mac Studio) have long had reliability issues with their HDMI output and certain monitors, during the boot process. Apple used to actually initialize the display before the OS was up, but they gave up doing that entirely with macOS 12.0 (on desktops). However, since we don’t yet ship a proper display controller driver with Asahi Linux, and since we need to be able to show m1n1, U-Boot, and grub boot screens anyway, we need to initialize the display in m1n1 to set up a boot-time framebuffer. This works fine, but the problem is that absent a display driver running actively throughout the boot process, any hot-plug events will cause the display to be lost, as DCP (the Display Controller Processor) shuts it down pending a new modeset command. Unfortunately, it seems a subset of monitors have a strangely broken behavior where, when waking up from standby, they will virtually disconnect their inputs a few seconds later, then connect them again. m1n1 would initialize the display, move on, the monitor would trigger a fake hotplug event, and DCP would shut down the signal.

And since Asahi Linux does not yet ship a proper DCP driver in the kernel, this meant a completely broken display (this will be fixed soon, but we still want bootloader graphics to work…). There were some test patches to solve this by introducing a long boot delay to let the monitor fully stabilize before continuing, but that’s an imperfect solution, and we didn’t like the idea of delaying boot for everyone, or forcing users to decide whether their monitor needs the workaround at installation time.

I dug around trying to find some solution to this problem, perhaps a DCP command to stop it from reacting to hotplug events, but I came up dry. And then I had an idea. Can we… shut down DCP? Back when we started reverse engineering DCP, we knew we couldn’t properly reset it after a crash, and it wasn’t clear how shutdown modes work. It felt like DCP had to stay up, and letting it fully go down would break everything. But we’ve since improved our understanding of how RTKit power modes work, how to correctly shut down coprocessors, and how to get them to come back up after a full power-down (which we use with e.g. NVMe). So I tried the new, known-good, full shutdown procedure… and it worked! With DCP down, there is nobody to react to the monitor hotplug event. The HDMI signal just sticks around, oblivious to whatever the monitor is doing. And since the DCP shutdown happens in a controlled manner, Linux can bring it back up without any issues for the real driver, once available.

There is still a minor race (if the monitor hotplugs between when m1n1 sets the screen mode and when it shuts down DCP), but it’s a tiny window. In practice, this new approach has eliminated the display issues for everyone affected, as far as we know. This update also makes it possible to unplug or power down your monitor while using Linux, without losing signal the next time you power it back on. Just update your system to get the new m1n1!

While we’re talking about DCP, Apple devices have recently been the target of an [advanced malware](https://googleprojectzero.blogspot.com/2022/06/curious-case-carrier-app.html) that used DCP to compromise the system. When we developed the (still not shipping) Asahi Linux DCP driver, we already did so treating DCP as being potentially compromised and hostile; in addition, we also do not expose DCP directly to userspace in a way that would render it exploitable. Rest assured, this vulnerability does not affect Asahi Linux, even though we use the same vulnerable DCP firmware as macOS/iOS.

## DART diversions

The M1 Pro and M1 Max/Ultra also introduced a new DART (IOMMU) hardware variant, which they use for the Thunderbolt ports and some of the media codec related hardware. R had reverse engineered this already, but there was no proper driver yet. It turns out the M2 uses this IOMMU variant throughout, so we added m1n1 and Linux support for it during the M2 bring-up process. This won’t make anything new work on M1 Pro/Max on its own, but it is a requirement for Thunderbolt and the media hardware, so that’s one more box ticked.

On the same subject, Jannau submitted a patchset to add support for the other DART variant in Pro/Max upstream, that splits off the IOMMU pagetable code into a dedicated file for Apple DARTs. We’ve long since had support for these chips in the Asahi fork, but upstreaming was blocked due to the upstream maintainers not being entirely happy with adding Apple-specific page table quirks to the ARM IOMMU page table code. We hope that by moving this to a separate file, we can unblock the process and finally get M1 Pro/Max support upstream (at least enough to boot on the chips, as upstream Linux already does on the M1; there are still other drivers left to upstream for full functionality, of course).

## U-Boot updates

U-Boot support has been progressing nicely too. U-Boot 2022.07 was just released with (almost) complete support for the M1 models (including M1 Pro/Max/Ultra) and (thanks to Janne Grunau) basic support for M2. The most important bit that's still missing is support for the PCIe USB3 controller behind the type-A ports on the Mac mini.

## More than Linux

OpenBSD 7.1 was released on April 21, 2022, with support for the M1 and M1 Pro/Max/Ultra. This includes support for NVMe, USB (USB 2.0 speeds only on the type-C ports), WiFi, Ethernet (on the M1 mini and iMac) and Keyboard and touchpad (on the laptops). X11 works on the initial framebuffer and audio support is being worked on. OpenBSD does not support the new M2 models yet, but this is on the radar for OpenBSD 7.2 which is scheduled for release in November.

OpenBSD depends on the Asahi installer, which provides the option to just install m1n1 with U-Boot as a payload. That gives you a standard UEFI boot environment, just like on other hardware supported by OpenBSD/arm64. At that point you can simply write minirootXX.img or installXX.img to a USB drive, plug it into your Mac and boot into the OpenBSD installer.

The OpenBSD installer knows about the magic Apple partitions, so even if you choose the default (W)hole disk option in the installer, the system partitions will survive and your macOS installation will be safe. The OpenBSD installer also picks up the WiFi firmware automatically (from the EFI system partition, where the Asahi installer prepares it), so WiFi is available during installation.

## One more thing…

I know what you’re all thinking… what about the GPU?

{{< tweet 1537828477352615936 >}}

Good news! A couple months ago, [Asahi Lina](https://twitter.com/LinaAsahi) joined our team and took on the challenge of reverse engineering the M1 GPU hardware interface and writing a driver for it. In this short time, she has already built a prototype driver good enough to run real graphics applications and benchmarks, building on top of the existing Mesa work. The proof of concept uses m1n1 via a USB connection and runs the driver remotely, so it is bottlenecked by USB bandwidth, but she has also demonstrated that the GPU proper renders the GLMark2 phong shaded bunny scene at over 1000FPS, at 1080p resolution. This fully open source stack passes [94%](https://twitter.com/LinaAsahi/status/1546301886147731457) of the dEQP-GLES2 test suite. Not bad!

But that story definitely deserves its own article, so we’ll be featuring a guest post by Lina telling us the whole GPU driver story in the near future. Stay tuned! In the meantime, you can follow her work on [YouTube](https://youtube.com/AsahiLina) if you’re interested.
