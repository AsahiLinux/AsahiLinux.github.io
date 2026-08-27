+++
date = "2026-08-26T20:30:00+10:00"
draft = false
title = "Progress Report: Linux 7.2"
slug = "progress-report-7-2"
author = "James Calligeros"
+++

Linux 7.2 has been released! That was fast. Let's dive in to yet another Asahi Linux progress report.
We've got a lot of interesting developments for you today, so make yourself a cuppa and enjoy.

## Think Different... again
The Apple Silicon platform's power management infrastructure is complicated. Responsibilities
are divided between multiple hardware blocks, including the SMC, PMGR, and PMP, all of which
have featured before on this blog. While supporting these blocks is important for power use,
one of the biggest obstacles to improving battery life has been the application cores themselves.

There are multiple ways to "sleep" a CPU core, and each one should be used in a specific context.
The most basic way to sleep an ARM CPU core is to use a Wait For Interrupt (WFI) instruction. This tells
the core to stop doing things until it is woken up by an interrupt from an interrupt source.
While this does save power by virtue of stopping the core from executing any code, the core
stays powered up and retains enough state for it to resume work extremely quickly. As such, WFI
is typically only used for parking a core on a running system. Apple cores include a "deep" WFI
mode, which shuts down more of the core at the expense of losing its state. Our downstream cpuidle
driver operates by setting WFI up in this mode, saving the core's state, then issuing a WFI loop.

Vendor-specific power management oddities like this are quite common. Thankfully for kernel maintainers, there
is a standard way to deal with them: the Power State Coordination Interface. PSCI
defines a standard iterface that allows an operating system to call into a defined set of
CPU core power management functions implemented by system firmware, including to prepare
them for sleep.

To avoid a proliferation of vendor-specific power management hacks inside the Linux kernel,
the maintainers of the arm64 arch-specific code have mandated that all upstream hardware
*must* use PSCI for power management. As such, we are not able to upstream our Apple-specific
cpuidle driver. Why are we still using it then?

PSCI defines "conduits" through which calls to firmware are to be dispatched from the kernel.
The two currently supported conduits in the kernel are the SMC (Secure Monitor Call) and HVC
(Hypervisor Call) instructions, which are used to yield execution to a higher Exception Level.
The Linux kernel expects to be running at EL2, which means that its PSCI calls must yield to
firmware running in EL3. Except Apple's cores do not implement EL3...

With the kernel already running in EL2 and no firmware running in EL3 to talk to, we are a bit
stuck. Linux cannot issue an SMC or HVC instruction since there is no EL3 to yield execution to,
which means we cannot make use of PSCI. Being able to properly power-manage the CPU cores is vital
for battery life and efficiency, so the status quo simply will not do. One quick and dirty solution
would be to have m1n1 load the kernel into EL1, and host a PSCI implementation in EL2. While this
would theoretically work, it would also break a lot of architectural features, such as virtualisation.
There must be something else we can do...

If you think about it, m1n1 is almost like our own firmware for Apple Silicon. mBoot (formerly iBoot)
starts it in EL2, it does its job, then jumps to whatever payload is attached to it. m1n1 does not
reserve any memory for itself and does not have any code that must stay resident, so its payload
is free to reclaim and overwrite that memory.

On production Asahi Linux systems, m1n1 loads U-Boot rather than the kernel directly. We do this
to make use of U-Boot's UEFI implementation, allowing distros and users to utilise whichever
standard UEFI bootloader (GRUB, systemd-boot, etc.) they want. UEFI also provides another feature,
Runtime Services. Much as the BIOS interrupts of old did, UEFI Runtime Services provide a way
for the operating system to access code originating in system firmware.

Reading the PSCI standard as published by Arm, one will notice that it deliberately defines
the API without reference to any specific conduit, and only lists SMC and HVC as examples.
If we take a broad interpretation of this, we could conclude that this means *other* conduits
are allowed by the spec...

To this end, Sven has been working on implementing a UEFI Runtime Service based PSCI conduit.
With m1n1's memory region carved out like other firmware regions, this will allow the kernel
to call back into it for PSCI services, even though it is running at the same Exception Level.
Sven has already modified m1n1 to reserve its memory and leave behind a PSCI implementation,
and the patches to the kernel enabling its use are already on the mailing list as an RFC!

## Please stop Thinking Different
Given that the cpuidle situation saw no progress until very recently, one might assume that
some event has catalysed work in this space. One would be correct.

The ARM specification mandates that cores in WFI loops should preserve all state. This is not
the default mode on Apple Silicon. On M1 through M3 series SoCs, state retention can be configured
on a per-core basis using chicken bits.

Due to a number of reasons that are not worth mentioning, Apple now sets each core's chicken
bits in mBoot and then locks down the registers controlling them starting with the M4 series.
This makes our life a little easier as m1n1 now has marginally less work to do, however it also
means that we cannot fine tune low level CPU behaviour. This is an issue on M4 particularly, as
calling WFI causes the core to lose its state and crash whatever was running on it.

Yureka noticed this while doing M4 bringup work, and added a kernel command line parameter
to make idle loop behaviour configurable. The parameter allows us to tell the kernel how
it should park cores in idle loops, including by doing a basic no-op loop. This prevents
M4 machines from crashing during early kernel initialisation, before our cpuidle driver
has been loaded. Once the driver takes over, it saves the the lost state before issuing WFI.
The patches to enable this are already in linux-next.

## They won't stop Thinking Different
Apple takes its reputation for platform security very seriously. As such, a lot of engineering
effort goes in to features that make exploiting vulnerabilities in their ecosystem infeasible
for all but the most sophisticated of attackers. One such feature is the Secure Page Table Monitor,
or SPTM.

Traditionally, the operating system kernel has been directly responsible for managing memory. This
includes handling memory allocations, virtual mappings to physical addresses, and MMU/IOMMU management.
A vulnerability in the code responsible for these operations could give an attacker access to
the platform's entire address space. In other words, you're cooked.

Naturally this makes memory management code a very common target for attackers, and given that
its job is fundamentally insecure (applications may want arbitrary allocations for just about
anything) it is incredibly difficult to lock down correctly.

Many years ago, Apple introduced the Page Protection Layer to XNU. PPL uses Apple's hardware security
features to isolate pagetable management from the rest of the kernel at the hardware level. This
works very well, however attackers eventually caught up and found ways inside PPL that allowed
them full access to the system.

Apple has had similar issues with IOMobileFramebuffer in the past. As mentioned on previous
blog posts, Apple (mostly) solved the IOMFB problem by putting it inside DCP's firmware,
behind an IOMMU. Nothing in macOS userspace nor in XNU can reach it, except via a defined
set of IPC functions. Taking inspiration from this approach, PPL eventually morphed into
SPTM. SPTM takes PPL and places it inside Apple's Guarded Execution Framework (GXF), a set
of Exception Levels that run parallel to the standard ARM64 Exception Levels.
GXF also comes with SPRR, a custom pagetable permissions system used when the CPU is running
code in GL1 or GL2.

When a modern Apple Silicon device starts up, iBoot or mBoot will detect if the configured
boot payload is an XNU image. If it is, it will first load SPTM into GL2. There, SPTM
sets up pagetables and memory management and locks down control of said features to GL2.
XNU then runs, using yet another IPC protocol to talk to SPTM. If XNU cannot successfully
contact SPTM, it will panic very early in init and halt the system.

This is problematic for us. If we try to run XNU under the m1n1 hypervisor, it will crash
as SPTM has not been loaded. If we configure mBoot to treat m1n1 as an XNU binary, m1n1
will crash as it will be unable to manage memory in the way it needs to.

Given that SPTM is mandatory for XNU on M4 and above, the hypervisor *was* rendered entirely
nonfunctional on these machines. We don't know when to quit however, which is why that's
*was* and not *is*!

SPTM itself is not particularly special. It is an ARM64 Mach-O binary that lives inside the
same preboot directory as the OS-specific firmware blobs for the various coprocessors. That
means we could theoretically have m1n1 load it _already under the hypervisor_, load XNU
next to it, then watch _both_ of them! But SPTM _must_ be run inside GL2 with SPRR
enabled. If only we [knew](https://blog.svenpeter.dev/posts/m1_sprr_gxf/) how both of
them [worked...](https://asahilinux.org/docs/hw/cpu/sprr-gxf/)

Thanks to Sven's work reverse engineering SPRR and GXF way back when Asahi Linux was first
getting started, he was recently able to teach the m1n1 hypervisor how to emulate them!
This allows us to load Apple's SPTM blob in exactly the manner XNU expects, do some quick
surgery on the XNU binary, load it, and then monitor MMIO accesses just as we could from
M1 to M3! The gory details of how Sven achieved this means that tracing is slower on these
machines, however not unusably so. This will allow us to continue bringing up new hardware
for the forseeable future!

## More M3 progress!
Work has also been progressing on bringing Asahi Linux to M3 series machines.

The webcam image signal processor remained largely unchanged, except for a single skipped initialisation
message on the M3 Max specifically. chaos_princess added support for that to the Linux driver,
and with that enabled full webcam support on all M3 series devices with a builtin webcam.

The builtin microphones also changed slightly. New to M3 series devices is a "High Frequency"
decimator requiring a new set of coefficients and a much larger initialisation message. Once
again, chaos_princess managed to get this figured out in no time, bringing microphone support
to all equipped M3 series devices.

ATCPHY --- the hardware block responsible for negotiating USB3, DisplayPort and Thunderbolt connections
over USB Type-C ports --- also changed slightly, requiring a new sequence of tunables at
initialisation due to the shift to TSMC's N3 process node.

While our thesis that Apple would avoid making sweeping architectural changes just for the
sake of it has *mostly* held true, we have of course run into a few such changes...

All M1 through base M3 series devices used an Apple-specific Texas Instruments USB port controller
called CD3217 (or ACE2). This sits on the I<sup>2</sup>C bus and negotiates with USB devices
when they are plugged in. Starting with M3 Pro/Max, Apple has switched to ACE3. ACE3 instead uses the
SPMI bus, which required some more reverse engineering. Thanks to the combined efforts of
[mildsunrise](https://github.com/mildsunrise/) and chaos_princess, we discovered that ACE3
has pretty much the same register set as CD3217, only wrapped in a SPMI interface instead
of addressed over I<sup>2</sup>C. Both the SPMI interface and ACE3 itself are now working
in Asahi Linux, bringing USB 3.0 and Thunderbolt support to all M3 series devices.

A massive change we *were* expecting was the firmware ABI for both the GPU and display
controller. Since the firmware for both AGX and DCP are paired with a specific macOS
version, Apple does not need to worry about keeping the interface stable across releases.
This is a large reason why we "target" specific macOS versions for each generation of
hardware. M3 series machines will target the ABI found in macOS 14.8.3, and support for
DCP is now almost at feature parity with the existing macOS 13.5 ABI we use for M1 and M2!

With all of this progress coming on top of milestones already reached on M3, we are pleased
to announce that we are almost ready to cut an official release! We will have more to say
about this in the coming weeks, so stay tuned!

## Don't forget about M4 and M5 too!
Not content with helping out only with M3, Yureka has also been working on M4 and even early M5
bringup. Beyond the WFI issue on M4, these SoCs suffered from a breaking change in Apple's NVMe
controller firmware in the macOS 15.x firmware bundle. Yureka and Sven worked together on
investigating the changes and implementing them in both m1n1 and Linux, so we now have working
NVMe on M4 and M5! Yureka also managed to get PCIe working to a state where devices on the
bus can be enumerated by Linux, and fixed an issue that caused Linux to crash shortly after
boot when more than one CPU core is enabled. Not much else is working on M4 and M5 so we aren't quite ready
to enable them in the Asahi Installer just yet, but as always we will have more to say in due course.

## Desktop video is hard
Last time, we announced preliminary support for the Apple Video Decoder, or AVD. This hardware
block accelerates H.264 (AVC), H.265 (HEVC) and VP9 video decoding on M1 and M2, in addition
to AV1 decoding on M3 machines and above. Since then, AVD support has been further refined by
sofus, with AVC, HEVC and VP9 all working mostly reliably on all machines supported by Asahi Linux!
We are now at the stage where we are considering desktop integration, which is where things
start to get tricky...

The AVD hardware is fundamentally stateless, meaning that all it does is take an encoded frame
and turn it into a video buffer. The hardware does no bitstream parsing, no decoding session
tracking, or any other management of the decoding pipeline. This lends itself well to the
V4L2 Stateless API, which is designed with such decoders in mind. While this API first landed
in the upstream kernel over 10 years ago, userspace has been slow to adopt it outside of specialised
software targeted at embedded devices. GStreamer has basic support for V4L2 Stateless, however
FFmpeg (and everything that consumes it) does not without out-of-tree patches. Desktop-class
software, such as web browsers, have historically focused on VA-API, NVDEC and VDPAU. More recently,
effort on the desktop has begun shifting to Vulkan Video. This has left V4L2 Stateless effectively
abandoned before it even got started for desktop software.

All hope is not lost, however. VA-API is now near universal in desktop-class software owing to
its adoption by both AMD and Intel for their video acceleration hardware. To prevent V4L2 Stateless
hardware from being unusable with such software, Bootlin authored a VA-API to V4L2 Stateless
translation layer. Sadly this has been abandoned for quite some time and no longer builds without
patches, however sofus has forked it and gotten it into a working state for AVD. With this
translation layer installed and an environment variable set for the login session, software
implementing VA-API support is now able to use AVD to accelerate video decoding! This does not
yet ship by default in Fedora Asahi Remix, and will not work with Firefox's video decoding sandbox,
however we hope to have something shippable soon.

## The road to direct scanout
The traditional benefit of hardware-accelerated video decoding has been the reduction in CPU
load. While this alone is of course hugely important, there are other places where power
and time is being wasted during video decode. Firstly, the CPU must copy the decoded frame
data to GPU memory. The GPU must then composite that frame into the scene as directed by
the compositor. The final rendered scene must then be copied to the display controller's
memory, and the display controller programmed to scan out the scene. This must all happen
at the playing video's framerate.

We can do better than this. Linux's DMA subsystem supports sharing memory regions between
multiple devices. In this context, it allows us to eliminate the multiple copies of the
same framebuffer data from one region to another. This only works if all hardware blocks
in the chain support the same framebuffer format, however. This is where things start
to get tricky.

Framebuffers come in all sorts of formats and sizes. Video data for example is
almost always stored and transmitted in a [semiplanar](https://en.wikipedia.org/wiki/Planar_(computer_graphics)) [Y'CbCr](https://en.wikipedia.org/wiki/YCbCr)
format, inspired by the analogue video storage and broadcast standards of the 20th century. On top of this,
graphics hardware almost always uses some form of specialised addressing. Pixels are not
"next" to each other in memory and are instead [tiled](https://en.wikipedia.org/wiki/Z-order_curve)
to reduce cache misses and other performance issues. Moreover, framebuffers are often
compressed too, further reducing memory footprint and pressure on the bus.

All of that to say, eliminating framebuffer copies between hardware blocks is not
simple. Each hardware block must agree on the framebuffer's pixel format, its addressing
mode, and any compression applied to it. If they cannot agree on this, copies and conversions
must be done in software.

One might be inclined to ask why such "shared" framebuffers can't just be created
with "raw" pixel data. Consider a 1920x1080 framebuffer containing raw 8-bit ARGB
pixel data. That is around 8 MiB of data. At 4K, this balloons to just under 32 MiB.
Even without the added penalty of copies, reading and writing this much data at 30,
60, 165, or even 240 frames per second is an _enormous_ strain on the memory bus and therefore
an enormous power drain. Even if we somehow had an infinitely performant and efficient
memory bus, each hardware block in the chain would still have to have enough local cache
or RAM off the bus, as well as the horsepower to process such a volume of data. This
is infeasible.

Luckily hardware vendors agree, and most "families" of display hardware solve this problem
by implementing support for the same addressing and compression schemes. This allows
their 3D engines, display controllers and video accelerators to share memory-efficient framebuffers
without having their own massive local caches or constantly hammering the memory bus.
On Linux, each driver declares which formats it supports and various parts of the software stack
then negotiate and agree on a common format to use for shared buffers. Thankfully for us,
Apple chose not to Think _too_ Differently here. Well, mostly.

Apple hardware uses two main addressing and compression formats. AGX, named after
the GPU, and Interchange.
While the AGX format is used exclusively inside AGX itself, the Interchange format is also
supported by DCP and AVD. On macOS, this enables the direct scanout of framebuffers from
both AVD and AGX, allowing Quartz (the macOS compositor) to opportunitically choose to
do almost nothing in a lot of everyday scenarios. If video is playing and nothing else is
happening, the GPU can shut off completely and all the compositor has to do is make sure DCP
is displaying new frames as AVD generates them. If a game or application is in fullscreen,
the compositor can simply pass DCP the address of that application's own framebuffer and
disable all of its own rendering activities. The good news is that we are now
well on the way to being able to replicate this in Asahi Linux!

Thanks to groundwork laid by Alyssa and Lina, the AGX driver already had basic support for
Apple's Interchange framebuffer format, it was simply not yet wired up. Oliver Bestmann and I
volunteered to wire up and enable Interchange support in the DCP kernel driver, as well as
both the Asahi and Honeykrisp Mesa drivers. Coupled with work already done bringing Y'CbCr
and overlay plane support to DCP, this has cleared the way for direct scanout of AGX-rendered
framebuffers to DCP! AVD support is on the way, with sofus currently investigating its ability
to output Interchange format buffers.

While this is all fantastic news, we are not quite ready to ship this to users just yet.
Kwin, the compositor for KDE Plasma, considers our setup "multi GPU" as AGX and DCP are
distinct hardware blocks. As such, direct scanout using the DMA-BUF API is completely
disabled for now. Kwin devs are actively working on improving this situation, and
direct scanout on setups like ours could be enabled in Kwin from as soon as Plasma 6.8!

## Sundry upstreaming
As always, we continue to get patches upstreamed. Progress has slowed in recent months,
and it may even appear as though we've gone backwards. This is only because we have
already upstreamed most things! Most of what is left is either enormous work that
we _cannot_ upstream yet (e.g. GPU, DCP) or _new_ features that we have only been able
to work on due to clearing most of the upstreaming backlog. That said, we still have
some odds, sods, bits and bobs sitting in the tree. Recently upstreamed patches include
a number of I2S peripheral changes related to enabling speakersafetyd to function,
the Devicetree patches enabling the SMC-based hwmon driver, and of course fixes for
all of the SMC firmware interface changes introduced by macOS 27.

## Thanks again!
Frequent readers will have noticed a number of new names popping up in the blog posts recently.
Thanks to the generous support of our [GitHub Sponsors](https://github.com/sponsors/AsahiLinux) and
[Open Collective](https://opencollective.com/asahilinux) backers, we have been able to get these
folks the hardware they need to work the magic that they have been. AVD support, M4 bringup, and
other goodies we have in store are only possible because of you.
