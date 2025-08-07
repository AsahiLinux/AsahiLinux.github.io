+++
date = "2025-08-07T22:00:00+10:00"
draft = false
title = "Progress Report: Linux 6.16"
slug = "progress-report-6-16"
author = "James Calligeros"
+++

With Linux 6.16 now out in the wild, it's time for yet another progress report!
As we mentioned last time, the Asahi and Honeykrisp Mesa drivers have finally
found their way upstream. This has resulted in a flurry of GPU-related work, so
let's start there.

### No missing nuts in this Flatpak
For quite some time, we have maintained a version of our Mesa driver built against
the Flatpak runtime and shipped it as a Flatpak runtime extension. This was required
to enable GPU acceleration on Flatpak while our Mesa driver was not enabled upstream.
As Flatpak runtime version 23.08 reaches end of life this month, this will no longer
be necessary. 24.08 now ships with Mesa 25.1, and so will the forthcoming 25.08
runtime. Once 25.08 is released, we will be discontinuing the Flatpak runtime
extension as it will serve no purpose.

### A _fully_ upstream graphics stack
Asahi uses DRM Native Context to paravirtualise the GPU. Rather than having to
virtualise high-level APIs like DirectX and Vulkan, the VM runs the userspace component
of the host's native GPU driver. The host is then passed native GPU commands that it
only needs to forward on to the kernel without modifying. This improves GPU
performance inside muvm, our virtual machine for [gaming on Asahi Linux](https://asahilinux.org/2024/10/aaa-gaming-on-asahi-linux/).
Until now however, this has relied on downstream patches to virglrenderer and Mesa.

We are pleased to announce that our DRM Native Context implementation has now
been merged into the upstream virglrenderer project, and will therefore be enabled
in upstream Mesa 25.2! This completes our transition to a fully upstream graphics
stack, and as such we are retiring our Mesa fork completely.

### GPU go brrrrr
Now that our Mesa driver is totally upstream, work on improving it has been able to
occur rapidly. Alyssa has been hard at work over the past couple of months optimising
performance. [A whole mess of merge requests](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/?sort=created_date&state=merged&label_name%5B%5D=asahi&author_username=alyssa)
have found their way upstream, with the bulk of them being merged in time for the
release of Mesa 25.2. Performance improvements are variable but decent, especially
when paired with [this month's improvements](https://fex-emu.com/FEX-2508/) to FEX's JIT.

### A fine vintage
Also new in Mesa 25.2 is support for `VK_EXT_map_memory_placed`, courtesy of
chaos_princess. This Vulkan extension allows applications to request GPU memory
be mapped at a specific address. WINE uses this when thunking 32-bit DXVK to
ensure any memory allocated by the GPU driver is within a 32-bit address space.
This is required to enable GPU acceleration for 32-bit Windows applications.
What's that? Your 32-bit games already work in FEX? Well...

The stack we currently ship to enable gaming is quite a beast. At the bottom
we have muvm, which spins up a micro VM (who would've guessed) using a kernel
configured with 4K memory pages. FEX as it ships today requires this; the x86
architecture only supports 4K pages and we cannot "split" a memory page into
smaller pages, only combine them to create pages of a larger size. Atop that
is FEX, which translates x86 binaries so they can run on 64-bit ARM devices.
But binary applications are very rarely self-contained monoliths. They require
access to kernel functions via syscalls, system libraries like glibc, and
access to the GPU via APIs like Vulkan.

The obvious solution---and the one used by FEX---is to translate the entire
world. When FEX loads an x86 or amd64 application, its loader pulls whichever
libraries the application binary is linked to from a disk image or mount point
containing a multilib amd64 root filesystem, and translates them. The benefit
here is that the application we want to run gets ABI-correct, native versions of
any libraries it's linked against. The disadvantages, however, are significant.

The image needs to have whichever libraries any arbitrary application could
realistically be linked to. We either accept that some applications simply will
not run, or we ship everyone an enormous filesystem image full of libraries for
a foreign machine that they will likely never use. There are also the runtime
costs associated with translating a bunch of libraries and faking syscall
interfaces for their benefit. We can do better than this.

The emulation stack is being used overwhelmingly for gaming via WINE/Proton.
For the sake of simplicity, let us simplify the problem space and say that our
goal is not to run x86 Linux applications, but x86 Windows applications, and
games especially.

Users of Windows on Arm will have noticed that it can run x86 applications.
I'll spare you the Windows NT history lesson for today, but the NT team's almost
religious focus on abstraction, portability and backward compatibility for nearly
40 years has given us three _very_ important and beneficial features:

- WoW64, which thunks out 32-bit system calls to their 64-bit counterparts
- An API to plug binary translators into WoW64
- 64K virtual memory pages, regardless of the underlying physical page size

Since WINE implements WoW64 and FEX implements the WoW64 binary translator API,
AArch64 WINE is capable of running 32-bit x86 applications. Since all Windows
applications are allocated 64K pages, we _theoretically_ (more on this in a bit)
don't need to worry about page size. With `VK_EXT_map_memory_placed`, all of the
pieces of the puzzle are now in place; AArch64 WINE can now run 32-bit Windows
applications and games _without_ muvm!

- We no longer have to paravirtualise the GPU, network access, or filesystem 
- We no longer have to mangle and proxy the X11 or PipeWire protocols
- WINE can now use Wayland directly rather than be forced through XWayland
- FEX only has to translate x86 Windows code; WINE calls out to native AArch64 64-bit Windows
  and \*nix 

That said, your mileage may vary...

While it's true that Windows allocates 64K virtual memory pages, not all applications
respect this. Windows application developers have a long tradition of simply
ignoring the Windows API, with game developers committing particularly heinous crimes
in pursuit of performance. This often means baking x86-isms like 4K physical memory
pages into code. Any applications which do this simply will not work, and will continue
to require the use of muvm. You will however be able to use WoW64 inside muvm, and these
applications will still reap most of the benefits from doing so.

You should also expect bugs in FEX and WINE, though unlike broken Windows
applications from 2003, these have a pretty good chance of being fixed.

Want to try this out for yourself? Some distros already make this seamless; thanks
to chaos_princess once again, Gentoo automatically installs FEX's WoW64
implementation on AArch64 systems when the `wow64` USE flag is enabled. Simply
install your preferred WINE flavour with the `wow64` USE flag and create a new
WINE prefix to get started!

On other distros, you will need to do some extra work:
1. Ensure your distro builds AArch64 WINE with WoW64 enabled. This is not currently
   the case for Fedora, but is coming soon!
2. Install WINE using your native distro packages
3. Download the latest `fex-emu-wine` build from the [FEX Ubuntu PPA](https://launchpad.net/~fex-emu/+archive/ubuntu/fex/+packages)
4. Extract `libwow64fex.dll` _only_ to`/usr/lib/wine/aarch64-windows/xtajit.dll`
5. Create a new WINE prefix

Once you have a WINE prefix all set up, you can start enjoying your x86 Windows
applications in a much improved fashion!

<figure>
    <a href="/img/blog/2025/08/lsw.png">
        <img src="/img/blog/2025/08/lsw.png" alt="LEGO Star Wars (2005) running in native AArch64 WINE using DXVK" />
    </a>
    <figcaption>
        LEGO Star Wars (2005) running in native AArch64 WINE using DXVK
    </figcaption>
</figure>
<br>

What about amd64/x86_64/x64 applications, you ask? Well unfortunately there are some challenges
unique to making amd64 Windows code run on an AArch64 Windows environment which mean we don't support this style of emulation for amd64 Windows applications.

<figure>
    <a href="/img/blog/2025/08/npp.png">
        <img src="/img/blog/2025/08/npp.png" alt="AMD64 Notepad++ running in WINE's ARM64EC mode" />
    </a>
    <figcaption>
        AMD64 Notepad++ running in WINE's ARM64EC mode
    </figcaption>
</figure>

_...Yet._

### We're getting there...
As always, we have been hard at work upstreaming various bits and pieces.

Devicetree bindings for the GPU have been accepted and merged for 6.17, which allows
us to stabilise how m1n1 initialises the GPU and forwards information onward to
the kernel.

The SPMI controller driver, required for power management, landed in 6.16. The
USB-C mux chips used in M3 Macs and above are also connected via SPMI bus, making
this driver necessary for USB on those machines.

A bunch of audio-related patches have landed in 6.16 and are queued for 6.17. Some
of these patches rework the upstream I<sup>2</sup>S controller driver, allowing
the speaker amps to feed speakersafetyd the data it needs. This brings us very
close to being able to upstream everything required to support speakers upstream.

The core SMC driver has been accepted for 6.17, along with subdevice drivers for
its GPIO controller and reset controller functions. Combined, these allow the
upstream kernel to not only shut down and reboot Apple Silicon Macs, but also
enable the WiFi/Bluetooth module and USB Type A ports on machines with them. Also
in the pipeline for upstreaming is the hwmon driver to enable fan control and access
to the myriad hardware sensors present on these machines, the SMC HID driver to support
power button presses for suspend/resume, and the SMC RTC driver to read and update
the current system time.

With Linux 6.16, we also hit a pretty cool milestone. In our first progress report,
we mentioned that we were carrying over 1200 patches downstream. After doing a little
housekeeping on our branch and upstreaming what we have so far, that number is
now below 1000 for the first time in many years, meaning we have managed to upstream
a little over 20% of our entire patch set in just under five months. If we discount
the DCP and GPU/Rust patches from both figures, that proportion jumps to just under half!

While we still have quite a way to go, this progress has already made rebases
significantly less hassle and given us some room to breathe.

### Miscellaneous fixes
Now that we have some time to do so, we have managed to progress a number of
things that had been sitting on the backburner.

Plymouth, the system used to display a boot splash screen instead of TTY messages,
broke its automatic DPI scaling on 13-inch laptops at some point recently. We have
now been able to fix this.

A recent update to OBS broke screencasting on KMSRO systems implementing DRM
explicit sync. This was found to be caused by compositor bugs, and has been fixed
in [kwin](https://invent.kde.org/plasma/kwin/-/merge_requests/7941), and a merge
request has been raised for [Mutter](https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/4547).
As those fixes are not yet released and other compositors might have the same issue,
Janne added DRM syncobj support to the DCP driver in [asahi-6.15.8-1](https://github.com/AsahiLinux/linux/releases/tag/asahi-6.15.8-1)
as a workaround.

Development on m1n1 has also started picking up again. The release of
[1.5.0](https://github.com/AsahiLinux/m1n1/releases/tag/v1.5.0) marks the first new
release since February. A number of small fixes were able to be merged, including
a fix for boot time crashes caused by missing calibration data on Macs that have had
their WiFi/Bluetooth module replaced. We also made it easier for distros to
add a custom logo to m1n1.

### Conference talks
Davide and Neal presented at [Red Hat Summit](https://events.experiences.redhat.com/widget/redhat/sum25/SessionCatalog2025/session/1731519631980001Xort)
and [DevConf.CZ](https://pretalx.devconf.info/devconf-cz-2025/talk/P3TEBA/),
covering the ongoing work on Fedora Asahi Remix and introducing a CentOS Stream port
as part of their work in the CentOS Hyperscale SIG. This was also covered
at [CentOS Showcase](https://cfp.fedoraproject.org/centos-showcase-2025-07/talk/RRT8NW/) in July.

### Project merch
Want to show your love for Asahi Linux to the world? Now you can! Head over to
[HELLOTUX](https://www.hellotux.com/asahi) to buy official Asahi Linux merch. A portion
of each sale is donated to the project. Many thanks to HELLOTUX for facilitating this!

### Until next time!
Our next report should hopefully bring even more improvements and developments,
so make sure to stick around. As always, a warm thanks goes out to our supporters
on [OpenCollective](https://opencollective.com/AsahiLinux) and [GitHub Sponsors](https://github.com/sponsors/AsahiLinux),
without whom this would not be possible.
