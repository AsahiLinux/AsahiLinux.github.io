+++
date = "2025-10-24T10:30:00+10:00"
draft = false
title = "Progress Report: Linux 6.17"
slug = "progress-report-6-17"
author = "James Calligeros"
+++

Another upstream kernel release means it must be time for a progress report!

## The patches keep coming
As always, upstreaming continues. The highlight of this cycle is finally landing
the SMC core driver, after discussions on the mailing list going all the way back
to 2022! Along with the core driver, drivers for the SMC's GPIO controller and
reboot controller have also been merged. This allows devices which are already
upstream to reboot gracefully, and lays the groundwork for enabling WiFi and
Bluetooth upstream.

Also merged for 6.17 are a few Devicetree changes which lay the groundwork for
upstreaming GPU support. While these are ultimately a small piece of the whole GPU
puzzle, they are very important. The Devicetree schema determines how the GPU kernel
driver gets the information it needs from firmware to properly control the GPU. Now
that this is stable and the properties merged into the upstream Devicetrees, we
can be reasonably confident that the parts of the GPU driver that interact with
the Devicetree won't require substantial changes.

Work is also progressing on getting more SMC functionality upstream. The hardware
monitoring, lid switch and power button, and RTC drivers are all close to being
merged, with just a few minor issues left to address.

Apple Silicon SoCs place peripheral devices and IP blocks (like the display
controller) behind an IOMMU called DART. These act as a sort of firewall for peripherals,
preventing them from accessing memory addresses that they shouldn't. The DARTs on the
M2 Pro, Max and Ultra (T602x) SoCs are *mostly* compatible with those found on
the base M2, with one caveat: they support a larger address space. Supporting this
required implementing four-level pagetables in the DART driver. This change, along
with the Devicetrees for all M2 Pro, Max and Ultra based devices have now been merged,
and are available in v6.18-rc1!

And of course, there are the USB patches. It seems like just about everyone's
been following these on the mailing list, so it would be remiss not to mention
them. Anyway, moving on.

## m1n1's starting to Rust
Slowing down the kernel treadmill has enabled us to finally start taking a look
at other parts of the stack, and we've been making some important (and long
overdue) changes to m1n1.

For a while now, the recommended method for installing non-Fedora distros has
been to use the "UEFI Only" installer option, then boot the preferred distro's
Asahi installation media from USB. However, it has been quite some time since
this installation option was updated, and the m1n1/U-Boot/Devicetree bundle it was
installing was over two years old. A lot has changed in that time, and it turned
out that this bundle was not capable of booting modern Asahi kernels. Oops.

To fix this, Janne has added a [CI pipeline](https://github.com/AsahiLinux/m1n1/actions/workflows/installer.yml)
to the m1n1 repo that will automatically build the UEFI bundle for us. With
this step automated, it is now trivial to upload the resulting artefact to our CDN
and point the Asahi Installer to it. The update cadence should now be a lot more
frequent than biennially!

We have also released [m1n1 v1.5.2](https://github.com/AsahiLinux/m1n1/releases/tag/v1.5.2).
This release is fairly uncontroversial, with the main changes being adding
compatibility for the now-upstream USB and GPU Devicetree schema. However, this will
likely be our last m1n1 release that can be built without Rust.

We believe that it is important for such a critical piece of software to be as
maintainable, safe and logically sound as possible, especially given that it is
*supposed* to root the chain of trust in a hypothetical Secure Boot environment.
Certain components of m1n1 being written in C means that we currently cannot meet
any of those goals.

One such component was the Apple Device Tree handling code. While the ADT is
implicitly trustworthy data, the original C implementation was quite haphazardly
put together. This resulted in an unsound API based on libfdt that was neither
ergonomic to use nor a particularly good fit for the ADT data itself.

I was able to develop a sound, idiomatic Rust implementation and API with a little
help from our resident Rust expert chaos_princess, and bind that back to the existing
C API so that all our existing C code can make use of it. After a round of bug-squashing,
we found that there is no speed difference between the old and new implementations,
even with the overhead of the FFI glue logic!

We don't have any plans to actively port *all* of m1n1 to Rust. Safety-critical
components in the boot chain are our primary focus, followed closely by components
which could directly benefit from a rewrite due to unsound or unergonomic C APIs.
Perhaps one day we will have an all-Rust m1n1, but if it happens it will do so
organically over time.

## WoW!
Last time, we teased working 64-bit Windows applications running in WINE outside of
muvm. Granted, Notepad++ is not a particularly exciting teaser, but we believe it is
important to be transparent and set realisitic expectations. Anyway, here are both
Hollow Knight and NieR:Automata---64-bit Windows games---running on WINE outside of muvm.

<figure>
    <a href="/img/blog/2025/10/hollowknight.jpg">
        <img src="/img/blog/2025/10/hollowknight.jpg" alt="Hollow Knight (2017) running outside of muvm on an M1 Pro MacBook Pro with Gentoo" />
    </a>
    <figcaption>
        Hollow Knight (2017) running outside of muvm on an M1 Pro MacBook Pro with Gentoo
    </figcaption>
</figure>
<br>

<figure>
    <a href="/img/blog/2025/10/nier.jpg">
        <img src="/img/blog/2025/10/nier.jpg" alt="NieR:Automata (2017) running outside of muvm on an M1 Pro MacBook Pro with Gentoo" />
    </a>
    <figcaption>
        NieR:Automata (2017) running outside of muvm on an M1 Pro MacBook Pro with Gentoo
    </figcaption>
</figure>

Thanks once again to chaos_princess, the toolchain changes we were waiting for last
time have now been integrated into Gentoo, and the required FEX DLLs are built
automatically when installing WINE.

Mileage will vary to an even greater degree than 32-bit applications, however.
WINE's support for ARM64X and ARM64EC (Microsoft's ABI and executable format
that allow AArch64 and amd64 code to interoperate) is still incomplete, so
many applications simply will not work (yet). We of course encourage folks who
like to live dangerously to give it a try, but we cannot provide any support
for this stack. Consider it an early alpha, a glimpse at better things to come.

## Helping developers
When we first started the project, we created [macvdmtool](https://github.com/AsahiLinux/macvdmtool)
to puppeteer a device under test using Apple's USB-PD Vendor Defined Messages.
These allow the user to reboot a Mac over USB and to route the SoC's primary
UART to the USB port. These features are very useful when doing early device
bringup. Rebooting the machine with this method works even when the system is
locked up, and having access to the UART means that we can inspect any boot logs
without relying on an initialised display or USB stack. macvdmtool relies on IOKit,
a macOS system library. Having to use macOS for development is painful, but what
if we could achieve the same functionality with a Linux utility instead?

[tuxvdmtool](https://github.com/AsahiLinux/tuxvdmtool) is a reimplementation
of macvdmtool on Linux, made possible by changes to the USB-C mux chip driver
present in our downstream kernel. tuxvdmtool offers the same features as macvdmtool,
with the added benefit of allowing the "host" system to be an Apple Silicon device
running Linux! Having access to a familiar and fully-featured Linux development
environment helps make early bringup work just that little bit more accessible.
We hope that tuxvdmtool encourages more folks to experiment with low-level bringup
on Apple Silicon devices.

While the kernel patches required to make this work are not upstreamable, a future
release of tuxvdmtool will be rewritten to use the Linux kernel's builtin
[i2c dev interface](https://docs.kernel.org/i2c/dev-interface.html), which will
allow us to drop them entirely.

## Helping everyone
We have always strived to work directly with upstream projects on Apple Silicon
support where possible. This allows us to discuss ideas with those projects, get
advice on the correct way to implement Apple Silicon support in their software, and
in many cases even benefit the ecosystem more broadly. While this means we are
beholden to the relevant upstream's release schedule, it also means that we don't
have to worry about maintaining outdated forks, rebasing, backporting security
patches, and any of the other software engineering things that are not directly
related to supporting Apple Silicon devices. The past year has demonstrated
clearly why not just an an upstream-first, but upstream-*only* approach to
development is the only approach that makes sense long term.

But working upstream is not only for our own benefit. Sometimes, the work we
do can be of benefit to other users on other platforms. Well documented examples
of this include getting projects to [work properly](https://asahilinux.org/docs/sw/broken-software/#fixed-packages)
on machines with 16K pages, and the PipeWire and WirePlumber enhancements that
allow us to automatically apply DSP profiles to each device. These were done with
the direct benefit to others front and centre of mind.

Sometimes, however, it can be those other users who find our work and put it to
use of their own accord.

Geometry shaders are a type of specialised shader which can work directly on
polygons rather than vertices. Although now considered a legacy technique,
some applications use them for various tasks, such as culling triangles from
a scene based on some sort of filter condition.

Similarly, tessellation is a graphical technique that allows GPUs to programmatically
fill a surface with new polygons. This is used to implement adaptive LOD, highly
detailed water, and increase the fidelity of models without having to ship
large, high-poly assets.

Modern high-performance GPUs support both of these with additional hardware,
however this requires engineering effort as well as room in the design's power
and die area budgets. Mobile GPU designers aim to minimise all three of these
and more often than not end up emulating both geometry shaders and tessellation.
Apple's GPUs, being derived from PowerVR and originally designed for iDevices,
are no exception.

While Vulkan does not mandate support for geometry shaders for this reason, OpenGL
does from version 3.2 onwards, as does Direct3D 10 and up. Since we would very
much like to support OpenGL as well as Direct3D translation, we need to be
able to execute geometry shaders somehow.

We may not have explicit support for geometry shaders, we *do* have support for
compute shaders. With these, we can emulate full geometry shader and tessellation
support on the GPU itself and avoid a great deal of the performance penalty
associated with using the CPU.

Of course this isn't as performant as full hardware acceleration, but it does
at least allow us to advertise support for geometry and tessellation shaders to
applications. This in turn allows us to claim conformance with all modern OpenGL
versions, allows Vulkan applications to use geometry and tessellation shaders,
*and* enables the translation of Direct3D for games running under WINE/Proton!

Other mobile GPUs also have this problem. Arm's GPU designs for example do not
have hardware support for geometry shaders or tessellation either. 
Enter [`poly`](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/37914),
an effort by Mary Guillemard to convert our geometry and tessellation shader emulation
into shared code that any Mesa driver can use. This can then be integrated into
any Mesa GPU driver, enabling geometry and tessellation shader support without
duplicating the emulation code itself!

Be sure to keep an eye out for geometry shader, tessellation shader, and transform
feedback support in Panfrost, the open source Arm GPU driver, as well as in KosmicKrisp,
a conformant Vulkan 1.3 implementation for macOS layered on top of Metal.

## Life, the Universe, and Everything. Plus one.
Keen eyes may have noticed we've started building Fedora Asahi Remix
[dailies](https://fedora-asahi-remix.org/builds.html) targeting Fedora Linux 43.
The new upstream Fedora release is slated to come out later this month, and
you can expect the Remix to follow shortly.

## At the Akademy
Neal and Janne attended [KDE Akademy](https://akademy.kde.org/) in Berlin this year.
As the annual KDE community conference, this was an opportunity to ensure future work
in KDE aligns well with Fedora Linux and Fedora Asahi Remix. Neal
[gave a talk](https://youtu.be/CznIsJQQE9U?t=248) about the Fedora KDE Plasma Desktop
Edition there, and prominently discussed Fedora Asahi Remix as part of it.

Additionally, there was a lot of discussion about the upcoming Plasma Setup application,
which will provide a better integrated experience for first boot of a Fedora KDE
install. Lessons learnt from Fedora Asahi Remix user feedback is being directly
incorporated into Plasma Setup, and Neal is working on integrating this for Fedora
Linux 44 as [a formal Change](https://fedoraproject.org/wiki/Changes/Unified_KDE_OOBE).

## M3? Kinda...
Way back in the early days, we predicted that Apple would refrain from making
sweeping architectural changes to the various functional blocks found in Apple Silicon
SoCs unless absolutely necessary, and that this would enable the reuse of a
lot of code and significantly cut down on the amount of reverse engineering required
between generations. The example we gave was the interrupt controller, which is
quite unfortunate because that was the first thing Apple ended up changing. Oops.

But the overall principle has held mostly true. So much so in fact that some
enterprising folks on Reddit have found that with only a trivial amount of
hacking, you can actually boot Asahi on M3 series devices!

It may be surprising to learn that very basic, low-level support for M3 has
existed for quite some time now. m1n1 is capable of initialising the CPU cores,
turning on some critical peripheral devices, and booting the Asahi kernel.
However, the level of support right now begins and ends with being able to boot
to a blinking cursor. Naturally, this level of support is not at all useful for
anything but low-level reverse engineering, but we of course plan on rectifying
this in due time...

If you're interested in helping us out and have prior experience with reverse
engineering or low-level embedded software development, we'd love for you to say
hi on IRC or Matrix! See [this page](https://asahilinux.org/contribute) for more
information.

## Thanks!
As always, a warm thanks to our sponsors on [OpenCollective](https://opencollective.com/asahilinux)
and [GitHub Sponsors](https://github.com/sponsors/asahilinux), without whom we
would not be able to keep the lights on. You're all truly wonderful!
