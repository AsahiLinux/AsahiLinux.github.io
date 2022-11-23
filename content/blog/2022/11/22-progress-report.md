+++
date = "2022-11-22T23:50:00+09:00"
draft = false
title = "Updates galore! November 2022 Progress Report"
slug = "november-2022-report"
author = "marcan"
+++

Time for another overdue progress report! This month's update is packed with new hardware support, new features, and fixes for longstanding pain points, as well as a new bleeding-edge kernel branch with long-awaited support for suspend and the display controller!

If you're new to Asahi Linux, check out our [previous release announcement](https://asahilinux.org/2022/03/asahi-linux-alpha-release/) for installation instructions and general information.

## USB3, Chapter 1

Until now, Asahi Linux has only supported USB2 on the Thunderbolt ports. While the hardware USB2/3 controllers are reasonably well supported by Linux already, and the Type-C port controllers are also based on existing partially-supported hardware, there was one big missing piece: the PHY driver.

M1 and later Apple Silicon machines use Apple-designed (or Apple-customized?) PHY hardware called "Apple Type-C PHY" (ATCPHY) that supports USB3, DisplayPort, and TB3/USB4 modes. This piece of hardware is in charge of turning the USB3/DP/TB protocol data into signals on the wires. Since we're dealing with very high-speed signals (up to 20Gbps per pair), the PHY has to be very complex and there are a lot of analog knobs that need to be individually calibrated. With USB2, you can get away with having universal settings that work for every device, but that won't work for USB3 and other higher-speed protocols!

The job of the PHY driver is to configure the physical hardware with settings specific to your particular chip, which are calibrated at the factory, and to manage reconfiguration of the entire PHY hardware as different modes are switched in and out. In practice, this means a huge number of "magic" register pokes, including some with variable data that comes from factory-written eFuses. [Sven](https://social.treehouse.systems/@sven) has been working hard reverse engineer all of this, and this new release includes his new ATCPHY driver with support for USB3 mode!

In addition to driving the PHY itself, the PHY driver has to very carefully coordinate with the USB controller driver (`dwc3`) and the Type C port controller driver (`tipd`). When devices are connected and disconnected, there is a complex dance of negotiation that has to happen that eventually leads to a decision on what protocols to run over which wires. This information has to be communicated to the PHY (including things like what orientation you plugged the cable in) so it can route its signals appropriately, and only after everything has been initialized in the right order can the USB controller be brought up. To make matters even trickier, the hardware is quite temperamental and if anything goes wrong the controller is likely to just lock up or fail to work!

We think USB3 mode should be pretty solid, but you can expect some glitches when doing things like hotplugging devices quickly at this point. The good news is that, since mode-switching when you hotplug the cable involves resetting almost everything, any transient issues can usually be solved by just disconnecting and reconnecting the device. Actual USB3 operation should be solid once connected, but do let us know if you encounter any issues.

## USB3, Chapter 2x7 Gen3.1415

That covers the Thunderbolt/USB4 ports... but there's another bit missing to the USB story: the Type A and non-Thunderbolt Type C ports on the Mac Studio and iMacs! These extra ports (which exist in different combinations depending on exactly which hardware model you have) are handled by an ASMedia PCIe USB3 controller. This is just a standard USB PCIe xHCI controller that is already well supported by the standard driver in Linux, but there's an Apple twist to the story: the controller needs firmware uploaded by the driver on Apple Silicon platforms!

Normally, this kind of hardware is expected to ship with Flash memory containing firmware to run the controller. However, Apple are not a huge fan of random Flash memories, and for good reason: they can be compromised, corrupted, and are difficult to verify and implement secure boot for. For this reason, Apple engineers have been steadily decreasing the number of firmware Flash memories on these platforms, and in this case, they decided to simply ship without it and have the kernel upload the firmware on startup.

This might seem like a simple problem, and indeed the actual firmware upload mostly is (though the undocumented register dance required to do this was not that easy to figure out and make work reliably...), but it ended up having us completely rethink the way we handle firmware in Asahi Linux, as well as making changes to our installer and shipping new software packages and vendored binaries!

As it turns out, the ASMedia firmware is embedded inside the macOS XNU kernel binary, so we need to extract it in order to make it available for Linux. We knew this was going to come up, so we've been stashing away a copy of that kernel in /boot/efi/asahi/ for future usage in our installer. But the kernel is in img4 format, and compressed with the Apple-developed lzfse algorithm, so we ended up having to ship this as a new package to allow existing Asahi users to seamlessly upgrade and have the firmware be extracted automatically.

Meanwhile, the same process has to happen on macOS inside the Asahi Linux Installer, for new installs. macOS of course ships with its own compression framework that supports lzfse, and we can use it directly from Python with `ctypes`... except there's another catch! It turns out that while this works on macOS, recoveryOS does not ship with `libffi`, which Python needs to make `ctypes` work. So we ended up having to vendor a copy of `libffi` (from the Homebrew package) with the installer, in order to make this all work seamlessly regardless of whether you are running the installer from macOS or recoveryOS.

As if that weren't enough, this firmware also made us completely redesign the vendor firmware mechanism in Asahi. See, until now we've been getting away with only loading firmware from the booted OS, once the initramfs has done its job. This works well enough for WiFi and Bluetooth firmware, but people usually expect their keyboard to work inside an initramfs, and that keyboard might be connected to one of the ASMedia xHCI ports. On top of that, the mechanism to install vendor firmware that we originally designed was racy, in that the WiFi/BT hardware could be discovered by the kernel before the firmware was ready on first boot (this is why sometimes you'd get broken WiFi on the first boot after install, particularly on M2 machines). It was time for a major change.

The new mechanism is described in the updated [Open OS Ecosystem on Apple Silicon Macs](https://github.com/AsahiLinux/docs/wiki/Open-OS-Ecosystem-on-Apple-Silicon-Macs#firmware-provisioning) page, but here's the gist: We changed the firmware package format from a tarball to a `cpio` archive, made the initramfs load it before anything else into a tmpfs, and then use a tmpfs mount on the final root filesystem to hold it. This also involved adding a new firmware load path to the kernel in a patch. The end result is a much more robust and even legally safer system, since the initramfs can ensure that the firmware is loaded before udev starts up (and therefore before any drivers could load that require it), and it is never persisted inside the root filesysem nor mixed together with distro-provided firmware (e.g. linux-firmware), which means clean root filesystem backups do not contain non-redistributable blobs. Even further, since the archive is a `cpio`, this allows the bootloader to load it as an add-on initramfs on boot, which means it can work with built-in drivers and completely eliminates all possible race conditions. While we are not doing this by default in Asahi Linux now, it is a supported configuration if you re-configure your `/boot` to directly mount the EFI system partition and install your kernels and bootloader there (we'll provide instructions on how to do this in the future; for now you're on your own, but the `asahi-scripts` package should seamlessly support both layouts). If the `cpio` was not loaded by the bootloader, the [initramfs script](https://github.com/AsahiLinux/asahi-scripts/blob/main/initcpio/hooks/asahi) will look for it and load it for you, so both configurations end up with similar results.

Phew! That was quite an adventure just to load some ASMedia firmware! But with that, first-boot WiFi is now rock solid on all machines, and we are much more confident in the firmware management design going forward.

## Audio Advances, Track ♪

What about the speakers, I hear you ask? We hear you! For months now we've had working speaker drivers, but we haven't enabled them for good reason: because we had the very strong suspicion that you could destroy your speakers without more complex volume limits and safety systems. As it turns out... those suspicions were correct! I decided to take one for the team and run some tests on my MacBook Air M2, and even with some sensible volume limits I quickly managed to blow up my tweeters. Oops! Good thing we haven't enabled the speakers yet!

This experiment led us to take a much closer look at what would be required in order to safely drive the 
speakers, and the answer is a lot more complicated than you might expect. Modern micro-speakers require sophisticated software EQ to sound good, but they also require sophisticated safety models! The most critical safety parameter for micro-speakers is the temperature of the voice coil: you don't want to melt the thing. But how hot the speaker gets depends on how much power is being driven into the speaker. Most audio sources do not have a ton of energy in the high bands, which means that tweeters usually do not receive much input power. But certain inputs (like a high-frequency square wave) can deliver maximum power into the tweeters, and quickly exceed their safety thresholds. Of course, you could limit the volume to a known-safe level, but this would result in significantly reducing the overall volume and leave a lot of unused headroom for most normal music and audio signals!

So what do we do? Modern speaker amps have the ability to measure the current and voltage across the speaker they are driving, and from this you can compute instantaneous power. Feeding this power into a model based on the physical characteristics of the speaker, we can estimate the voice coil temperature (and the magnet temperature, which is needed for a complete model), and impose volume limits if the temperature rises too high. This is [exactly what macOS does](https://github.com/AsahiLinux/linux/issues/53), and what we'll have to implement on Linux.

This kind of safety model is not new: it is already commonplace on Android phones, where it is usually implemented in DSP firmware. But of course, the desktop Linux ecosystem doesn't even have a speaker EQ database framework yet, nevermind safety models! The eternal lagging behind of Linux audio strikes again. What's the plan? While this isn't settled yet, our current idea is to implement the safety model in a stand-alone daemon that captures the voltage/current feedback data from the ALSA device, and drives the mixer volume itself as as means of implementing soft power limits, together with some kind of "safety watchdog interlock" with the kernel that only enables higher volume limits when the daemon is active and running. This would keep the problem away from individual audio daemons and subsystems, which can directly open the output device and send data to it (though they won't be able to use hardware volume controls, but since the data format is 24-bit PCM, software volume control is perfectly acceptable here).

Apple started using this safety model on the M1 Pro/Max series of laptops, and the 13" M1 MacBook Air and Pro models do not implement it. However, since the speaker amps do support the feedback feature, we plan to develop and implement our own safety model for those too. It turns out that speaker voice coil temperature can be estimated from the fact that copper has a temperature coefficient, so its resistance changes with temperature. By feeding a pilot signal into the speakers and using the voltage/current feedback, we can compute coil resistance and see how the voice coil temperature rises for a given power input, and then use this data to derive the right constants for a new safety model. Because if we can outdo Apple at their own game, why not? (And we're also going to have much more confidence in the limits for these models if we do this research and implement it, as opposed to coming up with some guesswork soft volume limits.)

Thus, the speakers remain disabled for now, but we're getting there!

## Audio Advances, Track ♫

Meanwhile, [povik](https://github.com/povik/) has been hard at work on the audio subsystem! He has reverse engineered the new and completely undocumented CS42L84 headphone codec used in the 14"/16" MacBook Pro series, as well as the Mac Studio and M2 MacBooks. This means that we now have headphone jack support across the board! He has also been upstreaming much of this work to the Linux kernel and keeping our downstream patch count low.

In addition, we are now shipping an [`alsa-ucm-conf-asahi`](https://github.com/AsahiLinux/alsa-ucm-conf-asahi) package that integrates this properly into PulseAudio/PipeWire and similar audio servers, to make volume control and hotplugging seamless. This package will also be used with the upcoming speaker support, to make device switching work properly. In the meantime, all you have to do to make your headphone jack work out of the box is upgrade your packages and reboot! The hardware volume control should now be automatically mapped to your desktop's volume control without any manual settings.

While testing this and other audio changes, we ran into multiple PulseAudio bugs and regressions that resulted in a very poor experience for users (outputs that stop working after being idle or on hotplug, etc.). Our upcoming speaker DSP configuration is based on PipeWire anyway, and we found that simply replacing PulseAudio with PipeWire today fixes all the strange issues and otherwise seems to work flawlessly. Therefore, we're also adding PipeWire as a dependency to our `asahi-desktop-meta` package, which means that existing users of the Asahi Linux Desktop image will be prompted to remove PulseAudio and replace it with PipeWire when they upgrade. Embrace the future!

## Backlight, Book I

You've all been clamoring for it! Thanks to ChaosPrincess' work, we now have keyboard backlight support! This works seamlessly with KDE, so your Asahi Mac can now shine in the dark.

This was ChaosPrincess' first driver and reverse engineering effort, and I'm very pleased with the result. While this is a very simple driver, they did a very good job mapping out the unknowns of the hardware in order to make it look and feel like a proper driver, and not a reverse-engineered hack. This is something we strive for when we work on figuring out how hardware works, and it's always a pleasure seeing new folks picking up on the approach. To quote their experience reverse engineering:

> 1. run m1n1
> 2. poke at macos
> 3. bang head against desk a lot.
> 4. ??????
> 5. profit, sometimes

May our MacBook keyboards bring light to the darkness!

## Backlight, Book II

*But what about the display brightness?*

Ah, we're getting there! As you may know, up until now Asahi Linux has not had a display driver at all. Instead, we rely on the boot-time framebuffer set up by the bootloader (iBoot on laptops, m1n1 on desktops). This means that Linux just gets a chunk of memory and writes pixels to it – no VSync, no buffer flips, no mode changes, no DPMS, and no backlight control support. Since leaving the backlight on is a huge battery drain, we added a hack to support turning off the backlight by literally turning off the GPIO pin that powers it on, but needless to say that was not the best user experience (and many users have been baffled by having their screen turn off when they tried to turn down the backlight, since the now-somewhat-outdated Linux backlight interface has no way of telling userspace whether 0 means "lowest" or "actually off").

In order to support the display output properly, we need a driver for Apple's DCP coprocessor and its firmware. We've already [talked about DCP](https://asahilinux.org/2021/08/progress-report-august-2021/) in the past, and how cursed the interface is! Since then, [Alyssa](https://social.treehouse.systems/@alyssa) wrote a Linux kernel DRM KMS driver for DCP and [Janne](https://social.treehouse.systems/@janne) took over maintenance, and he's been steadily adding features, including brightness control support. With that in place, DCP is finally good enough for adventurous users to daily drive on M1 systems (M2 support is coming soon)!

This fixes a lot of longstanding missing features, including the ability to drive >1080p displays without bootloader hacks, brightness control, VSync (on Wayland), resolution switching, and more. It also paves the way for the (still incomplete) external display support (over Type C/DisplayPort) that Sven has been working on, and is a requirement for Lina's upcoming accelerated GPU driver. Progress!

However, it does come with some caveats: the driver is complex and likely to introduce bugs and regressions, and it may also reduce performance on some setups, since it is really meant to be used together with GPU acceleration (the simpledrm framebuffer driver has some software rendering optimizations that DCP lacks) and clients using the modern atomic-modeset and swap APIs, like Wayland compositors. It also has some limitations when used with legacy clients such as Xorg - in particular, there is no support for true VBlank interrupts, and it is unclear whether the hardware/firmware supports this at all. This breaks XFCE4's window manager with compositing enabled. For these reasons, we are not enabling DCP by default for all users, but instead making it an opt-in feature. How, you ask? Well...

## Living on the Edge

With new and bleeding edge drivers like DCP, we run the risk of regressing existing users' experience. While a lot of people have been eagerly waiting for these new features, many others are already daily driving Asahi with the existing kernels and would like their experience to remain stable. We do have a development package repository (`asahi-dev`), but it regularly contains broken packages and is not intended for end-users, and it is difficult to switch back and forth between it and the regular `asahi` repo (please don't use `asahi-dev`, really, and *please* don't tell others to use it).

Here's what we're doing instead: Starting with this release, we are shipping a new `linux-asahi-edge` kernel package, which contains drivers and configurations not quite yet deemed stable enough to push to all of our users. This package can be installed side by side with the existing `linux-asahi` package, and will become the default boot entry, but you can always fall back to the non-edge kernel using the advanced boot options in GRUB. To use it, simply upgrade your packages, install it, and update your GRUB config:

```
$ sudo pacman -Syu
$ sudo pacman -S linux-asahi-edge
$ sudo update-grub
```

Since this is a bleeding-edge kernel, we ask you to please not report issues with it as new bugs on our tracker (we already know about a lot of them!). Instead, please leave a comment on [this](https://github.com/AsahiLinux/linux/issues/70) tracker bug, after verifying that it is not a known issue mentioned elsewhere in the thread. We'll triage issues over time, as the drivers mature and eventually are promoted to the main kernel.

Note that `linux-asahi-edge` tracks the same kernel tree as `linux-asahi`, and we just change the build-time configuration to enable or disable certain features. In order to avoid breaking your system, if you have it installed, you must always upgrade both packages in tandem (just `pacman -Syu` is fine). Failure to do so could leave you with a device tree mismatch, and might leave one of the two kernels unbootable.

Those eager enough to click the previous link might have noticed something else mentioned there... yup, there's One More Thing™!

## Idle Ideations

Next to DCP, one of the most-requested features is "power management". That's really amusing, because we have had power management from the get go! CPU frequency scaling, device runtime PM (for select devices), hardware auto-PM... even prior to this release, users could already get 10+ hours of idle runtime. With DCP and proper display DPMS, that now goes as high as 30+ hours (powered on, screen off)!

Ah, but when people say "power management", what they usually mean is "suspend". See, ancient x86 platforms (where "ancient" means "everything prior to 2015 or so") *don't* have reasonable real idle power management like Apple Silicon Macs do. Instead they implement a global "suspend" or "S3" mode that turns off most of the hardware, leaving only RAM powered on, because that was the only way to not completely eat through a laptop's battery in a few hours on those machines.

Thankfully, SoC-based systems like smartphones have had much better power management for a long time now, and even Intel and Microsoft got on board and now support something they call "S0ix" or "Modern Standby". On platforms with true fine-grained power management, you don't need to "suspend" the system to save power. You simply leave the machine idle, and it already saves power!

Of course, "leaving the machine idle" can be a tricky thing without a bit of help from the OS. On Linux, the equivalent of S0ix is called "s2idle" (Suspend-to-idle), and it does exactly what it says on the tin: it goes through the motions of suspending the system, but then just leaves the hardware in an idle state instead of going through platform firmware to actually trigger a full-system suspend. Crucially, this freezes userspace and forces certain devices into low-power mode, which means that s2idle can *guarantee* power saving, where just leaving apps running wouldn't. Some people have reported high battery drain on Asahi Linux machines while idle, and this is almost always caused by poorly-behaved userspace causing a large number of wakeups or keeping CPUs busy. s2idle solves this issue!

s2idle does not require any special drivers or support, but it does require working (i.e. at least non-failing) suspend/resume support in drivers. For us, this was blocked on the WiFi chipset, which required a new mechanism to go into what it calls S3 suspend (confusingly named; it maps to s2idle here) on Apple machines that wasn't supported by the existing driver, and would cause the suspend process to error out. After implementing this and some other minor driver features, s2idle now works on Apple Silicon machines! This is now enabled in `linux-asahi-edge` kernels, so if you switch you should see suspend options magically pop up in your desktop environment.

While s2idle does work, it's in its infancy and we haven't debugged all driver issues yet. Here's what works:

* NVMe is shutdown
* WiFi goes into S3 mode
* Display (DCP) goes into DPMS (backlight & screen fully off)
* DARTs power gate & restore state on resume
* CPUs stay in shallow idle
* Some misc devices (i2c/spi/etc) power off
* Wakeup via power button or lid open

And at least these things are known missing or broken:

* No CPU deep/poweroff idle (also affects normal runtime, blocked on PSCI replacement)
* USB2/3 is broken (controllers are reset and you might need to re-connect devices on resume; *please don't suspend with mounted USB drives!*)
* Some misc devices not really suspended yet (e.g. keyboard/trackpad will buffer keystrokes and may break)
* No alternate wakeup sources (keyboard/mouse/etc)

We'll work through some of these issues over time, but we wanted to give everyone a chance to try s2idle in its current state! As mentioned earlier, please report any issues in the [linux-asahi-edge tracker issue](https://github.com/AsahiLinux/linux/issues/70).

Finally, Apple Silicon Macs *do* support a "true" S3-like suspend mode that puts the whole system into a deeper sleep state, and this is what macOS uses for suspend. While this may give bigger power savings, it is more complex to implement and depends on a few things we haven't worked out yet. For now, since there is still low-hanging fruit to decrease s2idle power consumption, we will focus on that going forward. In the future, we may implement "full" suspend if we find it offers significant benefits over properly managed s2idle.

## Installer Iterations

We've also been working on the Asahi Linux installer for new users. To keep a long story short, we've fixed a lot of the pain points in prior versions (like issues computing the safe limits for resizing macOS that cause resize errors), and we've also improved the wording of some of the prompts to make things clearer. Hopefully it should result in a much smoother and less confusing experience for everyone!

In addition, we've promoted M2 support to non-expert. We're committing to supporting the 12.4 firmware version on M2, just like 12.3 on the M1 family. As a reminder, when we speak of firmware versions, we're talking about the *OS-paired* firmware which the installer installs alongside Asahi Linux. This is unrelated to your macOS version, and the installer is compatible with all current macOS versions, from 12.3 to 13.x.

There are also two new installer features: it can recover some broken installations, and it can upgrade your m1n1 stage 1 version. Sometimes, users fail to follow the post-installation (step 2) instructions properly, and that can leave them in a boot loop. While this is easily fixed by shutting down the system and trying again, doing anything else (like selecting another boot OS) can leave the installation in a weird state that makes it hard to try again without wiping the partitions and re-installing from scratch. The installer now offers a new feature to repair an "incomplete installation", which essentially re-does the last step of the process and prompts the user to follow the post-install instructions again, without requiring a full re-install.

This same mechanism is also useful to recover from rare cases where Apple's macOS upgrade process messes up and breaks your Asahi Linux install by deleting or re-creating its Boot Policy. We've seen this happen very rarely (if you find a way to reliably reproduce it, please let us know!). Users can now rest assured that the solution is simple if this happens: just run the installer again (`curl https://alx.sh | sh` from macOS or recoveryOS) and select the repair option when prompted (the situation is identical to an incomplete installation).

The new m1n1 upgrade feature is used to upgrade your m1n1 stage 1 version. There are two stages of m1n1 in an Asahi Linux install, and the second stage is always upgraded via Linux packages and is the most critical to add new features. The first stage is installed from recoveryOS and cannot be upgraded from Linux, and its job is just to load the second stage. Since that is all it does, it rarely needs to be upgraded. However, this can sometimes be useful to fix rare bugs, and we've identified an issue that can cause persistent boot failures after an unclean shutdown in rare cases. The workaround is simple (just boot by holding down the power button and select Asahi again, which will clear the fault), but the new m1n1 version should fix this problem entirely, so we recommend upgrading. Simply run the installer as usual (`curl https://alx.sh | sh`) from macOS or recoveryOS, and choose the option when prompted. If you run it from the recoveryOS paired to your Asahi install (that is, holding down the power button while Asahi is the default boot OS, and choosing *Options*), it will even avoid the usual mandatory reboot step.

Finally, there's a new expert-only experimental feature to install m1n1 (proxy mode only) onto an external USB drive. As we've said many times, Apple Silicon machines do not support USB boot. However, Apple fakes this by copying the OS boot files (kernel, bootloader, firmware) from the USB drive into a reserved partition on the internal NVMe storage. The installer can now take advantage of this mechanism even as a third-party OS, and do a "fully USB" install of m1n1, without requiring any re-partitioning of internal storage. If you decide to try this, you can unplug the USB drive after the installation is complete, and amusingly the machine will continue to cold boot into m1n1 anyway until you select another boot OS from the boot picker (we told you, the USB boot thing is fake!). Such a USB drive isn't automatically portable to other Macs, but in theory using the installer's "repair install" option should allow you to make a foreign m1n1 USB drive bootable on a new Mac, though we haven't tested it yet and there will probably be issues if the machine isn't of the same type/model at this time.

Unfortunately, since m1n1 stage 1 has no USB host support, this won't get us true "fully external" Asahi Linux installs just yet... does anyone want to start working on a Rust USB stack and xHCI host controller driver? ;-)

## Epilogue

As usual, for existing users, you can get all the new features by just upgrading your system with `sudo pacman -Syu` and then rebooting. This handles migration to the new firmware mechanism and everything else. We want to keep things simple for everyone! This time, we also recommend running `sudo update-grub` to ensure your GRUB bootloader is up to date (this is not part of normal package upgrades), and do make note of the aforementioned optional but recommended m1n1 stage 1 update.

This is probably a good time to remind everyone to please keep your Linux installations up to date! Early in the macOS Ventura betas, we found an issue that would make Linux unbootable after the upgrade. We fixed this many months ago in a package update, but we still get users upgrading to Ventura today (or even the latest 12.x macOS releases, which also have the same issue now) and winding up with unbootable Linux installs because they never upgraded in that time. While these kinds of problems where a macOS upgrade catastrophically breaks Linux should be rare (increasingly so over time), and we do test betas to catch them early, we rely on users to keep their OS updated in order to avoid trouble. Please update! Arch Linux updates can get pretty gnarly if you haven't done one in a long time too, so that's all the more reason to do so.

Oh, ah, right, the GPU. Yes. How could we forget! Hey Lina? You're on the hook for a state of the GPU blog post next week. No excuses!

![Xonotic on an Apple M1](/img/blog/2022/11/xonotic.png)
