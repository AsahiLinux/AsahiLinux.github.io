+++
date = "2022-03-18T23:50:00+00:00"
draft = false
title = "The first Asahi Linux Alpha Release is here!"
slug = "asahi-linux-alpha-release"
author = "marcan"
+++

It's been a long while since we updated the blog! Truth be told, we wanted to write a couple more progress reports, but there was always "one more thing"... So, instead, we decided to take the plunge and publish the first public alpha release of the Asahi Linux reference distribution!

We're really excited to finally take this step and start bringing Linux on Apple Silicon to everyone. This is only the beginning, and things will move even more quickly going forward!

Keep in mind that this is still a very early, alpha release. It is intended for developers and power users; if you decide to install it, we hope you will be able to help us out by filing detailed bug reports and helping debug issues. That said, we welcome everyone to give it a try - just expect things to be a bit rough.

Behind this scenes, this release brings with it several future compatibility features. What this means is that users who install it will be able to keep up with all future improvements by simply upgrading their packages - no reinstalling necessary.

Ready to give it a shot? Make sure to update your macOS to version 12.3 or later, then just pull up a Terminal in macOS and paste in this command:

<div class="curlsh">
<pre>curl https://alx.sh | sh</pre>
</div>

Pay close attention to the messages the installer prints, especially at the end!

## System requirements

* M1, M1 Pro, or M1 Max machine (Mac Studio excluded)
* macOS 12.3 or later, logged in as an admin user
* At least 53GB of free disk space (Desktop install)
    * You need 15GB for Asahi Linux Desktop, but macOS itself needs a lot of free space for system updates to work, so the installer will expect you to leave 38GB of extra slack in macOS by default to avoid shooting yourself in the foot. For example, if you have 60GB of free space, you will be able to shrink macOS by up to 22GB by default, freeing up 22GB for the new Linux install and leaving 38GB of remaining free space in the macOS partition. If you want to disable this check, enable expert mode when prompted.
* A working internet connection
    * The installer will download 700MB ~ 4GB of data, depending on the OS you select.

If you use Time Machine, you may find that you do not have enough free space. This is caused by local Time Machine snapshots taking up the "free" space in your volume. You can learn how to delete these to truly free up space [here](https://appleinsider.com/articles/21/06/26/how-to-delete-time-machine-local-snapshots-in-macos).

## Installation

The installer is designed to be as self-explanatory as possible. You will see a series of prompts that guide you through resizing your macOS partition (if necessary) and installing your new OS. We expect users to install Linux alongside macOS at this point. The installer will not delete or affect your macOS installation, other than performing a live resize.

Once the first stage of the installation is done, you will have to reboot into 1TR mode (One True recoveryOS) in order to finish the install. Read the instructions that the installer prints carefully! Simply rebooting into the new OS won't work until this is done. You need to fully shut down your machine, then boot by holding down the power button until you see "Entering startup options", choose your new OS in the boot selector menu, and follow the prompts.

The installer provides these OS options:

### Asahi Linux Desktop

A customized remix of Arch Linux ARM that comes with a full Plasma desktop and all the basic packages to get you started with a desktop environment. It includes a graphical first-boot set-up wizard, so you won't have to dig around to change your settings or create your first user. No root password by default; use sudo to become root.

### Asahi Linux Minimal (Arch Linux ARM)

A vanilla Arch Linux ARM environment, with only the minimal support packages to integrate with the boot process and hardware on Apple Silicon machines. Arch users will feel right at home!

Log in as root/root or alarm/alarm. Don't forget to change both passwords! SSH is disabled by default for security reasons, so you'll have to enable it manually.

### UEFI environment only (m1n1 + U-Boot + ESP)

No distribution, just a minimal UEFI boot environment. With this, you can boot an OS installer from a USB drive and install whatever you want (as long as it supports these machines, of course)!

## What works

All M1, M1 Pro, and M1 Max devices are supported except for the Mac Studio.

* Wi-Fi
* USB2 (Thunderbolt ports)
* USB3 (Mac Mini Type A ports)
* Screen (no GPU)
* NVMe
* Lid switch
* Power button
* Built-in display (framebuffer only)
* Built-in keyboard/touchpad
* Display backlight on/off
* Battery information / charge control
* RTC
* Ethernet (desktops)
* SD card reader (M1 Pro/Max)
* CPU frequency switching

M1 machines only (no Pro/Max):

* Headphones jack (might be flaky)

Mac Mini only:

* HDMI output

Not yet, but coming soon:

* USB3
* Speakers
* Display controller (backlight brightness control, V-Sync, proper DPMS)

## What doesn't

Everything else, but notably:

* DisplayPort
* Thunderbolt
* HDMI on the MacBooks
* Bluetooth
* GPU acceleration
* Video codec acceleration
* Neural Engine
* CPU deep idle
* Sleep mode
* Camera
* Touch Bar

Note: on the 13" MacBook Pro, you can use Fn + the number row keys (1-9, 0, and the next two) as F1..F12 in lieu of the Touch Bar.

## Known bugs

* If Wi-Fi doesn't work, try toggling it off and on in the network management menu
* If the headphones jack doesn't work or only one channel works, try rebooting. There's a flakiness issue.

## Known broken applications

The Asahi Linux kernel is compiled to use 16K pages. This both performs better, and is required with our kernel branch right now in order to work properly with the M1's IOMMUs. Unfortunately, some Linux software has [problems running with 16K pages](https://github.com/AsahiLinux/docs/wiki/Software-known-to-have-issues-with-16k-page-size). Most notably:

* Chromium (needs volunteer to fix)
* Emacs (fix committed, not released)
* Anything using jemalloc (e.g. Rust installed via the official Arch repositories)
* Anything using libunwind (fix committed, not released)

Hopefully this release will help motivate upstream projects to fix these issues and properly support all ARM64 page sizes (64K, 16K, and 4K). 16K provides a significant performance improvement of up to 20% or so under some workloads, so it's worth it.

There is a category of software that will likely never support 16K page sizes: certain emulators and compatibility layers, including [FEX](https://github.com/FEX-Emu/FEX). Android is also affected, in case someone wants to try running it natively some day. For users of these tools, we will provide 4K page size kernels in the future, once the kernel changes that make this possible are ready for upstreaming.

## Further information

Visit our [docs wiki](https://github.com/AsahiLinux/docs/wiki) for more information! We could use more people working on documentation, so feel free to contribute too! Note that, since things have been moving very quickly, some of the docs may be out of date.

Recommended reading:

* ["When will Asahi Linux be done?"](https://github.com/AsahiLinux/docs/wiki/%22When-will-Asahi-Linux-be-done%3F%22)
* [Introduction to Apple Silicon](https://github.com/AsahiLinux/docs/wiki/Introduction-to-Apple-Silicon)
* [Open OS Ecosystem on Apple Silicon Macs](https://github.com/AsahiLinux/docs/wiki/Open-OS-Ecosystem-on-Apple-Silicon-Macs)
* [m1n1:User Guide](https://github.com/AsahiLinux/docs/wiki/m1n1%3AUser-Guide)

If you're a developer or reverse engineer, we have some really cool toys:

* [SW:Hypervisor](https://github.com/AsahiLinux/docs/wiki/SW:Hypervisor)

And if you're bored:

* [Trivia](https://github.com/AsahiLinux/docs/wiki/Trivia)

## Frequently Asked Questions

### I want to donate!

Thank you! Asahi Linux is developed by a group of volunteers, and led by marcan as his primary job. You can support him directly via [Patreon](https://patreon.com/marcan) and [GitHub Sponsors](https://github.com/sponsors/marcan). None of this would have been possible without your support! Donations like these allow him to dedicate most of his time to the project and also purchase test hardware.

Martin Povišer is working on audio support for Asahi Linux, and he also has a [GitHub Sponsors](https://github.com/sponsors/povik) page. Please support his efforts so that he can make Apple Silicon machines the best supported platforms for Linux audio!

### Can I dual-boot macOS and Linux?

Yes! In fact, we *expect* you to do that, and the installer doesn't support replacing macOS at this point. This is because we have no mechanism for updating system firmware from Linux yet, and until we do it makes sense to keep a macOS install lying around for that.

You can have as many macOS and Linux installs as you want, and they will all play nicely and show up in Apple's boot picker. Each Linux install acts as a self-contained OS and should not interfere with the others.

Note that keeping a macOS install around does mean you lose ~70GB of disk space (in order to allow for updates, since the macOS updater is quite inefficient). In the future we expect to have a mechanism for firmware updates from Linux and better integration, at which point we'll be comfortable recommending Linux-only setups. Of course, if you *really* want to wipe macOS after installing Linux and re-partition that space, we won't stop you!

### How do I switch between Linux and macOS?

Boot with the power button held down, then pick which OS you want to boot. You can select it while holding down the Option key in order to make it the default.

### Help! I'm in a boot loop!

You didn't read the instructions at the end of the install process, did you? :-)

You need to fully shut down your machine, wait 15 seconds, then press *and hold* the power button until "Loading startup options" appears. Then select your newly installed OS, and follow the instructions.

If holding down the power button doesn't bring up the boot picker but rather keeps rebooting, you might be in a rare failure case. This shouldn't happen any more, but just in case it does, don't despair. Your machine is fine. You need to wait until the Apple logo disappears at the end of a loop, then count down 3 seconds and *quickly press, release, and press and hold* the power button. This should bring up the boot menu again. Just make macOS the default and that will fix things. We don't expect anyone to run into this any more, but one user saw it with an early pre-alpha version of the installer after encountering a network error. We've made a change to avoid that, but we'd like to mention the solution just in case.

### Can I install offline?

The installer is designed to be online-only at this point. While you can run it offline if you have a copy of all the files it needs, we don't have a user-friendly mechanism for setting that up yet. Please let us know what your use case is so we can better support it in the future!

### Can I install to an external/USB disk?

Apple Silicon machines cannot boot from external storage. While it may look like they do when you choose an external macOS volume, behind the scenes parts of its boot components are being copied to the internal drive to make this work. It's unclear whether this mechanism will ever be usable by third party OSes, for technical reasons.

Instead, we recommend using the *UEFI environment only* installer option to install only a UEFI bootstrap to your internal drive. This only requires around 3GB of disk space, and it will then automatically boot from any connected USB drive with a UEFI bootloader. Note: installing the Asahi Linux desktop images to a USB drive automatically isn't supported right now, though if you're adventurous enough it's not terribly hard to do manually :-)

### How do I uninstall it?

The installer does not have an uninstall option at this time, but all you need to do is delete the partitions it created, e.g. using `diskutil`. You can then resize the macOS partition back to full size if you wish.

If you perform more than a couple installations and removals, you might find that some parts of the install process take longer than usual. This is caused by stale Boot Policy files being left behind. We have a [script](https://github.com/AsahiLinux/asahi-installer/blob/main/tools/cleanbp.sh) that you can use to clean those up. Use with caution; you need to run it from recoveryOS or with SIP disabled.

### What's with all the disk space?

Each OS needs around 3GB of disk space, plus the space required for the OS root/boot filesystems. 2.5GB of this is used for the "stub" macOS partition, that includes critical components such as Apple's bootloader and firmware, and a full copy of the macOS recovery image. This is required by the design of these platforms. We reserve 2.5GB (even though the recovery image is only ~1GB) to allow for upgrades and avoid trouble caused by overfull APFS filesystems.

### Is there GPU acceleration yet?

Nope! We're working on that and have been making steady progress behind the scenes. In the meantime, though, software rendering is surprisingly usable thanks to the M1 family's outstanding CPU performance. The Asahi Linux Desktop image sets up a full composited X.Org session, with transparency and shadows and all that bling, and it doesn't feel sluggish at all! Quite a few people are already daily driving these machines in this state.

Once GPU acceleration is available, existing users will be able to get it by just doing a package upgrade!

### Does audio work?

The headphones jack works on M1 models (M1 Pro and Max models have a new codec chip that isn't supported yet). We already have working drivers for the speakers, but there are safety (we don't want to blow up your speakers) and quality (we want it to sound *better* than macOS) issues that need to be worked out still, since support for automatic speaker DSP is still in its infancy in the Linux audio ecosystem. Therefore, we have not enabled them for end-user installs yet. Once this is ready, a simple package upgrade will be all you need to do to get your speakers working!

### Does sleep work?

Not quite yet. We have working s2idle including putting USB/NVMe to sleep, but the Broadcom Wi-Fi driver needs additional development to make it support sleep properly on these machines, and the PCI driver also needs some tweaks. Once those are ready, we'll enable s2idle support in our kernels.

Whole-system sleep will come later, but can also be enabled with a simple package upgrade thanks to the future-proofing work that is already in place.

### The notch!

That's not a question, but assuming you're wondering what we're doing about the notch: on M1 Pro and Max systems today, Asahi Linux crops the screen such that the notch is hidden, and the desktop environment sees a rectangular 16:10 display. We will continue with this approach for now, but in the future we will enable opt-in display modes that include the notch, so that desktop environments which implement notch-avoidance can request them and gain additional screen real estate.

### Is this just Arch Linux ARM?

Pretty much! Most of our work is in the kernel and a few core support packages, and we rely on Linux's excellent existing ARM64 support. The Asahi Linux reference distro images are based off of Arch Linux ARM and simply add our own package repository, which only adds [a few packages](https://cdn.asahilinux.org/aarch64/asahi/). You can freely convert between Arch Linux ARM and Asahi Linux by adding or removing this repository and the relevant packages, although vanilla Arch Linux ARM kernels will not boot on these machines at this time.

If you're curious, [these](https://github.com/AsahiLinux/asahi-alarm-builder/) are the scripts we use to build the Asahi Linux images starting with a vanilla Arch Linux ARM tarball. We also sponsor an ALARM mirror as part of our project (jp.mirror.archlinuxarm.org).

As part of our project, we are contributing fixes and improvements to a number of miscellaneous packages; for example, Apple GPU support for Mesa is already well on the way (it is being tested on macOS), and we also have a fork of xkeyboard-config where we will prototype keyboard layout adjustments to better support Macs before upstreaming.

### Can I install other distros?

Absolutely! Our goal is to work with other developers to bring full support for these machines to your favorite distro.

That said, at the time of this writing no other Linux distros provide official Apple Silicon install images to our knowledge, and vanilla ARM64 images will not work until the necessary changes have trickled into upstream kernels; therefore, setting up another distro is more of a manual process. Some users are providing [unofficial installation guides](https://github.com/AsahiLinux/docs/wiki/SW:Alternative-Distros) and images for other distros, and in the future we will add some of them as options to the Asahi Linux installer. Keep in mind that these are user-contributed and we cannot provide end-to-end support as the Asahi Linux project.

If you are a developer for a distribution and interested in officially supporting Apple Silicon machines, please get in touch! We'd love to work with you to make it happen. You can find us on our [IRC channels](https://asahilinux.org/community/).

We would also love to hear from non-Linux OS distributors. OpenBSD snapshots can already be installed after setting up an UEFI boot environment, see [here](https://www.openbsd.org/arm64.html) for more information!

### Will this break my machine? How safe is it?

We have strived to make this installer as safe as possible. All disk management operations are performed behind the scenes using native macOS tools (`diskutil`) and the installer doesn't really do anything truly dangerous.

In fact, we haven't seen any data loss or machine damage during the entire development of Asahi Linux, other than obvious user error ("I accidentally wiped my disk"). Apple Silicon machines are almost completely unbrickable: you can boot them in a special [burned-in recovery mode](https://support.apple.com/guide/apple-configurator-2/revive-or-restore-a-mac-with-apple-silicon-apdd5f3c75ad/mac) and recover them, using another machine connected via a USB cable. For those who don't have another macOS machine to act as a host, we have [open source tools](https://github.com/libimobiledevice/idevicerestore) that work on Windows and Linux too.

That said, as with all open source software, especially an alpha release like this one, we can't make any promises. It could eat your data. It probably won't, but don't blame us if it does.

While the installer tries hard not to let you do anything dangerous, there is of course nothing stopping you from wiping your macOS data once booted into Linux; you are, of course, in control of your own computer. Those who want to use disk management/partitioning tools should avoid touching the first and last partitions on the internal NVMe drive. Doing so could render your system unbootable and require a DFU restore. In particular, if you do not use the Asahi Linux images but instead boot a third-party installer using the UEFI environment, *never* choose "use the entire disk" when installing.

### What's with those scary security warnings?

During the second stage of the installation after the first reboot, the installer needs to put your new OS into "Permissive Security" mode in order to boot a third-party OS kernel. This only affects the OS that you have just installed; *your existing macOS install is not affected in any way*. Unlike Intel Macs and platforms such as Android, Apple Silicon Macs keep a separate security state for each installed OS. That means that there is absolutely no detrimental effect to macOS. You can continue using all of its high-security or DRM-related features, such as FileVault, iOS applications, Apple Pay, Netflix in 4K, etc. in tandem with a Linux install.

If you are concerned about data integrity/confidentiality in your macOS volume, we recommend enabling FileVault. This will make your macOS data entirely inaccessible to other OSes without logging in with your macOS credentials.

### Does installing this require phoning home to Apple?

All OSes installed on Apple Silicon machines require certain components provided by Apple. Since we can't redistribute these ourselves, the installer will download them from Apple's public CDN. In the future we will provide an option to allow caching this locally or to a USB drive, so you can do offline installs.

During the initial installation (when making the volume the default boot option), there is a process that runs behind the scenes that involves authenticating the new OS install against Apple's servers (from their point of view, this looks like authenticating a normal macOS install; they won't know you're installing Linux). This is part of the "Full Security" mode, which is transiently used for Asahi Linux before the install is switched to "Permissive Security". If this concerns you, you can launch the Asahi Linux installer from recovery mode (hold down the power button → Options → Utilities menu → Terminal). Doing it this way unlocks an alternate method that starts out as "Reduced Security", removing the phone home requirement, which the installer will automatically use. Note that it will still have to download boot components from the CDN though (but that does not involve any unique machine identifiers, it's just a plain HTTPS download).

### Do you support Secure Boot / Full Disk Encryption / etc.?

Apple Silicon machines are one of the few general purpose platforms that allows you to install your own custom OS while still maintaining a strong secure boot chain. Installing the bootloader requires physical access to the machine and your machine owner credentials (this is why we need to ask you to hold down the button to boot at the end of the install process!). Therefore, we are very interested in further supporting this in Linux in order to have a highly secure and attacker-resistant system, taking advantage of Apple's SEP, Touch ID, and more, while still retaining full user control over their OS. We are designing the Asahi Linux boot process in order to allow this in the future, but the necessary bits aren't ready yet, so please stay tuned!

In the meantime, we recommend that users concerned about physical security enable FileVault in macOS. This will implicitly add a log-in requirement to recovery mode, which will prevent an attacker with physical access from being able to compromise your OS that way. m1n1 does not yet have a secureboot mode, but it also doesn't have any local access features as long as it is installed properly. Locking down U-Boot and GRUB and the rest of Linux is left as an exercise for the user.

The Asahi Linux installer does not have an option to set up FDE for you. However, you can use the UEFI-only option and roll your own traditional LUKS setup manually. We expect that users interested in advanced secure boot and encryption set-ups will do their own thing, at least for the time being.

### I have another question

Drop by our IRC channel and ask away! You can visit us on #asahi at OFTC. Check out our [community page](https://asahilinux.org/community/) for more details.
