+++
date = "2026-06-30T20:30:00+10:00"
draft = false
title = "Progress Report: Linux 7.1"
slug = "progress-report-7-1"
author = "James Calligeros"
+++

Linux 7.1 is now here, and of course with it comes another progress report. We've got M3
progress, Apple bugs, and more!

## Welcome back Master Boot Record
When you long-press the power button on your Mac to bring up the boot picker (or use the
Startup Disk application), what you see listed as Asahi is not actually the partition
with the operating system on it. Apple's boot tooling will only work with what it considers
to be a "valid" macOS installation inside an APFS container. So that we can use Apple's
bootloader and avoid needing users to run commands from Recovery every time they want to
use Asahi, the Asahi Installer creates a small APFS container (2.5 GB) with just enough
of macOS on it to convince Apple's tools that it is a bootable installation of macOS with
m1n1 as its kernel. This arrangement worked completely unchanged from macOS 12 to macOS 26,
and Apple even fixed a couple of bugs in their tools that are only encountered when attempting
to boot raw binaries that are not a real XNU kernel.

Shortly after the release of the macOS 27 Golden Gate developer beta however, we began
receiving reports that people were no longer able to boot into Linux on their
machines --- the option had simply disappeared from both Startup Disk and the boot picker!
Obviously this is quite concerning, and so we made investigating this a priority.

Inspecting the disk using diskutil revealed that all Asahi-related partitions were still
present on the disk after upgrading to macOS 27. No data loss was occurring, which is a
positive sign. Additionally, Asahi was still bootable on the same machine when using the
boot tooling from a second install of macOS 26.

chaos_princess began inspecting Apple's own macOS Installer and old streams from way back
when we were first poking at Apple's boot tools. The macOS Installer sets some APFS
metadata before rebooting the machine, which further investigation revealed to be a flag
that marks the volume as bootable. Until macOS 27, the boot tooling simply ignored this
flag entirely. After setting the flag manually on an Asahi APFS container, it becomes
available in the macOS 27 boot picker with no further changes.

Going forward, all new Asahi installs will have this flag set automatically by the
Asahi Installer. We've also added an installer mode that will fix existing installations.
If you've installed the macOS 27 developer beta and cannot access your Asahi install, please
run the installer again and use the "Fix macOS 27 boot picker compatibility" option.

chaos_princess has also developed a program that can be run from Linux to fix the issue.
While we would eventually like to deploy this fix automatically, we need more testing
data to confirm that it is reliable and will not destroy anyones' filesystems. That's where
you come in. If you are willing to help us test this, clone [this repo](https://github.com/AsahiLinux/asahi-fix27),
then build and run it from Linux _before_ upgrading to macOS 27. If your Asahi volume
is still selectable as a boot target from macOS, it has worked. Do be sure to let us
know how it went by popping in to one of our [channels](https://asahilinux.org/community)
on OFTC or Matrix, especially if you run in to any issues.

## Three bytes forcing shutdowns
macOS 27 also brings firmware updates for all peripherals with global firmware, including
the SMC. One of the SMC's myriad functions is battery management. Our Linux power supply
driver talks to the SMC to get information such as charge state, voltage, time until empty and
battery health. The driver also uses the SMC's firmware interface to configure charge
start and stop thresholds to prolong the life of the battery. macOS 27's SMC firmware
changed one of the battery management interfaces from returning a 32-bit integer to
returning a single byte. This change confuses our driver, which under certain conditions
considers the battery as having failed and initiates an emergency shutdown to protect
the system. We have already patched this in the downstream kernel; starting with version
7.0.12, the power supply driver can deal with both firmware ABIs.

## On installing betas
Bugs like these are an important reminder that developer betas are just that, developer
betas. It is ill-advised to install them on production machines. The two issues we
have had so far have luckily minor, but that doesn't mean that all future issues will
be too. Global firmware updates are effectively permanent too, and can only be rolled
back with a DFU restore of the machine. Please refrain from installing developer betas
going forward. We have sacrificial machines we use to test these things on your behalf,
there is no need to risk your own expensive hardware and important data.

## The more things change...
Designing and validating computer platforms and the ICs that go into them is extremely
expensive and time consuming, so it makes very little sense to make changes to existing
designs when they are not necessary. Early on in the project, we made a bet that Apple
would agree and refrain from making constant breaking changes to either. Discounting a
few of the larger SoC blocks like the GPU that are almost required to change every
generation, this bet has largely paid off.

Audio on an Apple Silicon laptop involves a few different ICs and SoC blocks. The
defacto industry standard for audio ICs is I<sup>2</sup>S, an I<sup>2</sup>C-based bus
optimised for audio data. Apple's I<sup>2</sup>S controller has remained unchanged
since M1. All of these audio ICs also need a stable clock source, which must be
configurable to accommodate the wide variety of audio data rates. Apple's Numerically
Controlled Oscillator (NCO) has also remained unchanged since M1. Apple have also used
the exact same speaker and headset amplifier chips in almost all Apple Silicon machines.
So, when chaos_princess started adding speaker and headphone jack support to M3 machines,
little more was required than some trivial Devicetree additions and config files for
asahi-audio and speakersafetyd. As such, M3 machines now sport high-quality audio output
on Asahi Linux!

M3 machines have also grown support for both CPU frequency switching and proper big.LITTLE
task scheduling. Apple have not changed how CPU frequency switching works since the base
M2, meaning that all M3 and M3 Pro/Max/Ultra SoCs required nothing more than Devicetree
changes to work with our existing cpufreq driver. Tasks should now be more intelligently
placed on either efficiency or performance cores according to their requirements, and the
CPU cores themselves should clock up and down based on load. This will both save energy
_and_ improve performance!

Adding support for the SMC's hardware sensors was similarly trivial; the SMC's firmware is
not materially different across machines, so once again nothing more than a few Devicetree
changes was required here.

On top of the above, we also have PCIe, WiFi, Bluetooth, NVMe, keyboard, trackpad, and
other core SoC block drivers working in Linux for M3 series machines. Most of this work
has come by way of [Yureka](https://fedi.yuka.dev/yuka), who has been very busy hacking
on both m1n1 and Linux with her M3 series machines for a while now. We still have a ways
to go before we can start enabling Asahi Installer support for these machines, but progress
is rapid so watch this space!

## We're writing firmware now?
Most of the complicated hardware on this platform uses complicated firmware blobs. Most
of this is based on RTKit, an RTOS-like firmware framework used by Apple to present a mostly
standardised interface for the kernel to talk to the various bits of hardware. There are
exceptions to this, however. Some blocks, like DCP (Display CoProcessor) and
AOP (Always-On Processor), use RTKit as the basis for their firmware, but layer
yet another set of abstractions called EPIC on top of it. Others still,
like the Broadcom WiFi/Bluetooth chipset, use third-party firmware that Apple has no direct
control over. Then, there's the Apple Video Decoder (AVD).

AVD is special. Its firmware is neither RTKit nor EPIC, it's a secret third thing. The
hardware itself is essentially an ARM Cortex-M3 controlling a series of fixed-function hardware
units for decoding video frames encoded in AVC (H.264), HEVC (H.265), VP9, and AV1 on more recent
SoCs. The CM3 runs a blob of firmware that exposes an interface for XNU to point it to video
data, and then programs the actual decoder hardware itself. This would normally be fine, however
Apple made the interesting choice of bundling both the AVD's firmware and a pile of configuration
data _inside_ the AVD kext. Making matters worse, each SoC has a slightly different AVD variant.
This is logistically challenging, as the Asahi Installer would have to constantly be updated
with (and keep track of) Apple's changes to the offsets of the firmware data in the kext.
We _could_ do this, but what if there's a better way?

The firmware loaded by XNU is not verified by the CM3. It will begin executing from
its reset vector when signalled, no matter what is actually there. What if we
just... used our own firmware?

Since the firmware is effectively just there to abstract away the underlying video
decoder hardware, it doesn't actually matter what it does, so long as it installs
interrupt handlers for the various hardware blocks. If we understand what the underlying
hardware expects, we can just program it all from a Linux driver. To do this, we need
to understand how the firmware drives each decoder.

Being standard Cortex-M3 code, it is possible to run the AVD firmware in an emulator.
A number of solutions exist to do this, including QEMU, which allows you to single-step
your program and inspect bus and register operations. The groundwork for this was laid
many years ago by Jamie, R and Eileen, who through a combined effort managed to reverse
engineer the instructions and data formats required by the AVC and VP9 decoders. 

The XNU kext also applies a unique set of tunables to each AVD revision. We are not
entirely certain what these do, so applying these essentially needs to be a replay of
MMIO writes made by XNU. We need to keep track of each AVD revision, each set of tunables,
and which revision they need to be applied to. This would be impossible to maintain
satisfactorily in an upstream Linux kernel driver, so this should also probably live
in firmware.

While not much work happened on this front for a long time, new contributor [sofus](https://github.com/sofus13)
recently stepped up to fill the gap. With a blob of custom AVD firmware that simply
installs interrupt handlers and applies each variant's set of tunables, he was able to
write a working V4L2 driver for the AVC hardware! The hardware can decode 10-bit AVC-encoded
video up to 4K, and works well with software that implements the V4L2 Request API.
Keeping the firmware basic and stateless, with userspace and the kernel being responsible
for parsing all video data and programming the decoders themselves, also enables us to
more easily support other video acceleration APIs like VA-API and Vulkan Video at some
point in the future.

There's still some work to do before we can ship AVD support to users. AVD supports VP9,
HEVC and even AV1 on some SoCs, but we have not implemented support for any of these yet.
Some devices also have quirks that must be tested and accounted for in the driver. We
hope to have something shippable for you all in the not too distant future!

## A large m1n1 release
We have also recently tagged version 1.6.0 of m1n1. This is a consequential
release for distros, as it is the first version that requires Rust for stage 2
builds. Previously, m1n1 only made use of Rust when built with chainloading
support. Stage 1 m1n1 replaces the XNU kernel in Apple's boot tooling, and is used
only to mount the EFI System Partition and chainload Stage 2 m1n1 from there. A
little while ago however, we made the decision to move GPU initialisation into
m1n1. This removed the need for the kernel driver to deal with the floating point
numbers found in Apple's hardware initialisation data, and also greatly simplified
the Devicetree bindings. The version of the GPU driver we eventually submit to
the Linux Kernel Mailing List will therefore rely on m1n1 to do this initialisation
for it. We also ported the Apple Device Tree parsing code to Rust, which is
consumed by just about every other part of m1n1.

Given that m1n1 is effectively firmware, it uses `no_std` Rust and targets `aarch64-none-softfloat`.
To avoid pulling in superfluous dependencies, you can pass `BUILDSTD=1` to `make` to build `core`
and `alloc` without requiring a full `softfloat` toolchain to be installed.

Version 1.6.0 also brings a whole host of improvements to M3 series support, including
support for the SPMI controller and PCIe initialisation. We also now support tunnelling
the SoC's hardware UART directly over DebugUSB with [kisd](https://github.com/AsahiLinux/kisd),
which can be used to achieve much the same functionality as the Central Scrutiniser.
Much of this work is also courtesy of Yureka.

We are also laying the groundwork for M4 and A18 Pro (MacBook Neo) support, with better
handling of Apple's non-macOS boot mode and support for new power domain metadata found
in the Apple Device Tree.

## Thanks again!
As always, we would like to thank our generous supporters on [GitHub Sponsors](https://github.com/sponsors/AsahiLinux)
and [Open Collective](https://opencollective.com/asahilinux), without whom we would not be
able to continue working on unfinished M1 and M2 features or work on M3, M4 and A18 Pro
support while supporting our enthusiastic new contributors!
