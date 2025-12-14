+++
date = "2025-12-14T11:30:00+10:00"
draft = false
title = "Progress Report: Linux 6.18"
slug = "progress-report-6-18"
author = "James Calligeros"
+++

Linux 6.18 has released, and we have a bit to talk about for this holiday season progress report!

## More upstreaming madness
Last time, we announced that the core SMC driver had finally been merged upstream after three long
years. Following that success, we have started the process of merging the SMC's subdevice drivers
which integrate all of the SMC's functionality into the various kernel subsystems. The hwmon driver
has already been accepted for 6.19, meaning that the myriad voltage, current, temperature and power
sensors controlled by the SMC will be readable using the standard hwmon interfaces. The SMC is also
responsible for reading and setting the RTC, and the driver for this function has also been merged
for 6.19! The only SMC subdevices left to merge is the driver for the power button and lid switch,
which is still on the mailing list, and the battery/power supply management driver, which currently
needs some tweaking to deal with changes in the SMC firmware in macOS 26.

Also finally making it upstream are the changes required to support USB3 via the USB-C ports. This
too has been a long process, with our approach needing to change significantly from what we had
originally developed downstream. The changes made to the Synopsis USB controller and the TI USB-PD
controller drivers have been merged, with only the driver for Apple Type-C PHY (ATCPHY) requiring
review. ATCPHY is responsible for configuring the physical USB-C connection, detecting and negotiating the
required protocols and routing signals to and from the relevant controllers. This is required not
just for USB3, but also all the other protocols one could reasonably expect a USB-C port to carry...

## It's always a single line...
While most of us have been enjoying full microphone support, owners of M2 Pro/Max MacBooks have been
missing out. Given that the M2 Pro/Max is almost identical to the M1 Pro/Max, it was assumed that this
would extend to the "Always-On"[^1] Processor, which controls the microphone array. Unfortunately, it seemed
for a while as though this assumption would not hold. What could be the issue though? A bug in RTKit?
A change in how AOP is initialised? The only way we were going to find out was by buying a device...

Armed with a shiny ~~new~~ refurbished 16" MacBook Pro and a fresh Gentoo install, I started digging
and quickly ran into the first major issue; everything kinda works. The AOP is booted correctly, the
microphone endpoint is initialised, and the ALSA driver registers a PCM with the correct metadata.
Trying to open that PCM, however, fails silently. Oh dear.

Just how does one debug something like this? Perhaps you could run Linux as a guest of m1n1 and
stare at hexdumps for hours like we do for macOS. Or maybe you'd use the kernel's builtin debugging
features like ftrace or kgdb. Knowledge is leveraging all of the debugging facilities available to you
to quickly and efficiently pinpoint the exact point of failure. Wisdom, however, is `printk()`.

And so it was for the next eight hours. `printk()`, compile, boot. `printk()`, compile, boot.
Oops, followed the wrong branch. `printk()`, compile, boot.

Eventually, I ended up pretty deep in the DMA core, and came out the other side somewhere near
IOMMU code. A memory allocation was failing, and this was bubbling back up to the ALSA driver. Following
along just a little further revealed that the allocation itself wasn't failing, it was mapping of the
memory region to an I/O virtual address.

For a multitude of reasons, the IOMMU core supports imposing a limit on the range of valid addresses
for a given IOMMU. For DART, we specify these ranges in the Devicetree using the `apple,dma-range`
property. If anything tries to map an IOVA outside of the specified range, the driver will return
an error. The values for this property come from Apple's Devicetree, but have to be manually entered
into the Linux Devicetree source. You already know where this is going...

With the `apple,dma-range` property removed, one more boot finally gave us what we wanted --- working
mics on an M2 Pro/Max laptop! A patch was quickly applied to the kernel tree and a version tagged off.
Be sure to update your systems once your distro releases a kernel based on this tag!

We still have some work to do, however. Even with the _correct_ values in the `apple,dma-range` property,
our IOVA allocation still fails. Thankfully, this property is not actually required and isn't
specified for the AOP DART in any other SoC's Devicetree, so M2 Pro/Max MacBook users can at least
enjoy using their microphones while we figure this out.

Thanks to the funds made available by our generous supporters on GitHub Sponsors and OpenCollective,
we are able to afford not only to keep the lights on, but to procure the hardware we need to reverse
engineer and support the platform. That said, these funds are not an infinite money glitch.

OpenCollective works predominantly on a reimbursement model, where an invoice for a purchase
_already made_ must be presented and approved before funds can be released. This means that all of our
expenses are paid for out of pocket, with a delay of up to 5 weeks between making the purchase and getting
reimbursed. MacBooks are not cheap, and we do all still have other expenses in our lives. All of this to say,
we can't just magic up a device at a moment's notice and drop everything to look at every bug we encounter.
Things need to be prioritised, including our non-Asahi commitments. We understand that the wait for this
has been long and testing on everyone's patience, and we thank you for bearing with us.

## Asahi at the Congress
It is December, which means the Chaos Communications Congress is almost upon us. We are pleased to
announce that not only are some of us attending 39C3, but our very own Sven will be giving an
Asahi-themed [talk on Day 4](https://fahrplan.events.ccc.de/congress/2025/fahrplan/event/asahi-linux-porting-linux-to-apple-silicon).
Topics will include our reverse engineering process, what we've already achieved, and what the future
holds. Be sure to check it out, timezone permitting!

## A better installation experience
We've always said, somewhat tongue-in-cheek, that we want Asahi-the-distro to cease existing.
At its core, Asahi Linux has always been a reverse engineering project. An "official" Asahi distro,
be it Asahi ALARM or Fedora Asahi Remix, has always been considered a temporary measure while we work
with the various upstream projects to integrate our code. By now everyone is aware of our work with
the Linux kernel, PipeWire, WirePlumber and Mesa, but we also work with less obvious --- yet equally
important --- projects.

Apple Silicon Macs are effectively scaled up iPads, and their SSDs reflect this. These machines use the
SSD to store the bootloader, firmware, factory calibration data, and data used by the SEP for security.
Each machine also has a system recovery partition at the end of the disk, and a paired recovery
environment for each installed macOS instance. If _any_ of these are deleted, your data is toast and
you will need to DFU restore your Mac.

This is part of the reason that Fedora Asahi Remix is installed via a disk image. It is far safer
to allow our installer to sanely partition your Mac's disk and flash known-good images into the
relevant partitions. This approach is great for a single-distro pipeline, however comes with
some serious drawbacks.

Firstly, it is unrealistic and unfair to expect other distros to take on the release engineering
burden of building special disk images and installers for a single platform. Not only is this
a lot of effort for a relatively small user base, it also involves a lot of duplicated effort
across each distro.

Secondly --- and most importantly for a lot of our users --- this approach is rather unhygienic
from an information security perspective. Flashing a pre-rolled disk image makes it impractical
to set up disk encryption at install time, and means that we need to manually scramble partition
UUIDs on first boot.

In an ideal world, installing Linux on an Apple Silicon Mac should be as easy as using the Asahi
Installer to prep the machine, then booting your favourite distro's live image and installing from
there. Some distros, such as Gentoo, are already taking this approach. Gentoo can get away with
this because the partitioning is entirely manual and users almost always follow the documentation
step-by-step; a little guidance and care on the user's part is enough to avoid most partitioning
mishaps. However, distros with a graphical installer will typically offer to automatically partition
a user's disk with the distro's defaults. These tools are mostly designed around the PC, and will
happily wipe out the entire target disk. We obviously cannot allow this to happen, and the fact that
it is highly likely means that we cannot support a traditional, mainstream install approach. Yet.

Every partition on GUID Partition Table[^2] disk is marked with a partition type UUID. These are
standardised, and never change. Apple's special partitions are included in the standard, allowing
us to easily identify a Zone of Avoidance for partitioning operations.

To this end, Neal has been working with the Anaconda team to hide these partitions so that they
cannot be removed accidentally by Anaconda's disk partitioning module. Once this work is complete
and filters through to a release, distros which utilise Anaconda will be safely installable from
live media! The goal is for Anaconda to present _only_ the free space created
by the Asahi Installer, enabling fearless automatic partitioning. This also enables users to easily
specify custom disk setups, such as full disk encryption or nonstandard filesystems if so desired.

In a similar vein, Neal has also been working with KDE on Plasma Setup, a new onboarding application
intended to make system setup on first boot a pleasurable experience. Once stable, Plasma Setup will
allow us to replace the current Calamares-based first boot wizard with a modern, integrated experience!

## Happy holidays!
As 2025 comes to a close, we want to thank everyone who has supported us financially on [Open Collective](https://opencollective.com/asahilinux)
and [GitHub Sponsors](https://github.com/sponsors/asahilinux), contributed code, reported a bug,
or even just offered moral support! We will see you all up ahead.


[^1]: Not actually always on.
[^2]: Remember when this is what GPT stood for?
