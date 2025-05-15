+++
date = "2025-05-15T13:00:00+10:00"
draft = false
title = "Progress Report: Linux 6.15"
slug = "progress-report-6-15"
author = "James Calligeros"
+++

Linux 6.15 is right around the corner, which means it's time for another progress
report! We have been pretty busy behind the scenes and we have some exciting developments
to share with you all.

## Fedora Asahi Remix 42 Release
Last time, we announced that Fedora Asahi Remix 42 was close to release. It has since
[released](https://fedoramagazine.org/fedora-asahi-remix-42-is-now-available/) and is
now available to install! The Asahi Installer now offers Fedora Asahi Remix
42 images by default, and existing users on FAR 40 and 41 are encouraged to upgrade using
[`dnf system-upgrade`](https://docs.fedoraproject.org/en-US/quick-docs/upgrading-fedora-offline/)
or Plasma's Discover to start enjoying the latest versions of your favourite
software. This is also a reminder that Fedora Asahi Remix follows the [Fedora Linux lifecycle policy](https://docs.fedoraproject.org/en-US/releases/lifecycle/),
and thus Fedora Asahi Remix 40 is now fully end-of-life (EOL).

## Fewer forks, more spoons
We are pleased to announce that our graphics driver userspace API (uAPI) has been merged
into the Linux kernel. This major milestone allows us to finally enable OpenGL, OpenCL and Vulkan
support for Apple Silicon in upstream Mesa. This is the only time a graphics driver's
uAPI has been merged into the kernel independent of the driver itself, which was kindly
allowed by the kernel graphics subsystem (DRM) maintainers to facilitate upstream Mesa enablement while
the required Rust abstractions make their way upstream. We are grateful for this one-off exception,
made possible with close collaboration with the kernel community.

This means that we will soon sunset our Mesa, virglrenderer, and Flatpak runtime forks. Eliminating these forks lightens
our maintenance burden, and working directly with upstream Mesa improves the development
experience for folks working on the userspace graphics stack. It also means that other distros
like [Debian](https://salsa.debian.org/xorg-team/lib/mesa/-/commit/bcd9afe05d2e31459eb8c1f54b6dda2a257cbf14)
and [Gentoo](https://github.com/gentoo/gentoo/commit/23e382acf4f7d75e49bc694f409c92385283632f), and
the [Freedesktop SDK](https://gitlab.com/freedesktop-sdk/freedesktop-sdk/-/commit/13e0add938f4c74887a08ad0ef6493502a8d3913)
can provide userspace graphics support for Apple Silicon without any additional packaging burden.

Fedora Asahi Remix will drop the forked packages with the upcoming Fedora Linux 43
based release. We expect that to happen without user intervention. The transition will be
disruptive for Fedora Rawhide, but Rawhide is not supported or expected to be used on end-user systems.

Upstreaming the uAPI has been an ongoing effort behind the scenes for
a long time. Making a single change to the uAPI requires commensurate changes to the
kernel driver, Mesa and virglrenderer. These changes need to be synchronised, since Mesa
and virglrenderer rely on the uAPI to communicate with the kernel driver. Some keen observers
may have noticed that [many](https://lore.kernel.org/asahi/20250310-agx-uapi-v1-1-86c80905004e@rosenzweig.io/) [versions](https://lore.kernel.org/asahi/20250313-agx-uapi-v2-1-59cc53a59ea3@rosenzweig.io/) [of](https://lore.kernel.org/asahi/Z-Fn4niI6_Yd06Ze@blossom/) [the](https://lore.kernel.org/asahi/20250323-agx-uapi-v4-1-12ed2db96737@rosenzweig.io/) [uAPI](https://lore.kernel.org/asahi/20250326-agx-uapi-v5-1-04fccfc9e631@rosenzweig.io/) [were](https://lore.kernel.org/asahi/20250327-agx-uapi-v6-1-df6b878a61b2@rosenzweig.io/) [submitted](https://lore.kernel.org/asahi/20250408-agx-uapi-v7-1-ad122d4f7324@rosenzweig.io/) to the kernel mailing lists
before it was finally merged. Some of the changes between versions were fundamental
in nature, requiring significant rework to the userspace components and
kernel driver itself. Alyssa and Janne dedicated countless hours to this endeavour over
the past few months - making changes, testing them, changing the changes, testing the
new changes, rinse and repeat. As ever, they have our deepest gratitude and respect
for pouring so much of their time and effort into closing this out.

## Even more kernel upstreaming!
The past couple of months have also led to even more kernel patches finding their
way upstream. Linux 6.15 sees the introduction of the Apple Display Pipe (ADP) display
controller and Z2 touchscreen digitizer drivers, which together enable Touchbar
support in the upstream kernel for the M1 and M2 13" MacBook Pros.

Also finding their way upstream are a number of critical patches for supporting various
functional blocks in Apple's SoCs. PCIe controller support has been merged for the
T6020 SoC (M2 Pro), which lays the groundwork for supporting the USB-A ports on
the M2 Pro Mac mini, as well as WiFi and Bluetooth on all M2 Pro devices. WiFi/BT
additionally depends upon the System Management Controller. Work is ongoing to get
the SMC driver upstreamed.

Linux 6.15 also includes some patches we require for audio, particularly around the TAS2764 and TAS2770 speaker amp chips.
These patches add basic support for the Apple-specific variants found in Apple Silicon
Macs.

## I can't triforce
Last update, we released microphone support for most laptops. We have since added
support for the M1 and M2 13" MacBook Pros. Unfortunately, wider release of the microphone
stack revealed a number of issues. We discovered that the Always-On Processor (AOP) on M2 Pro/Max devices
differs slightly from the rest of the Apple Silicon family, meaning that microphones
currently do not work on laptops with those SoCs. Work is underway to sort this out,
so hang tight!

As discussed last time, Triforce is my naive attempt at implementing a beamformer with
minor prior knowledge. In my haste to get something working out the door in a
reasonable timeframe, I made some questionable, _temporary_ engineering decisions.
Surely I would have time to undo these soon, right?

There is nothing more permanent than a temporary solution. Life got in the way, and I
found myself with no time to rectify the issues. Ah well, performance is pretty
bad, but it works well enough...

One of the assumptions I made when building Triforce is that PipeWire's "quantum" (buffer
size) will always be 1024 samples. At the time, I thought this would be true on every Apple Silicon Mac. Turns out that was a
bad assumption. If Triforce sees an input buffer that is smaller than 1024 samples,
it will return nothing to the graph, effectively muting the microphone.

[Frédéric Bour](https://github.com/let-def) ran into this somehow on Fedora Linux 42, and in
the process of fixing it also took it upon himself to fix many of the other bad choices
I made in the course of development. The end result is that Triforce should now be more
accommodating of oddball PipeWire configurations, and about 4 times faster! A huge
thanks to Frédéric for picking up my slack on this one.

## Upcoming talks
May brings with it [Red Hat Summit](https://www.redhat.com/en/summit) and
June brings [DevConf CZ](https://devconf.info/cz). Asahi will be present at both.
At RH Summit, Neal and Davide will [present](https://events.experiences.redhat.com/widget/redhat/sum25/SessionCatalog2025/session/1731519631980001Xort)
Fedora Asahi Remix and CentOS Hyperscale Asahi Remix as accessible platforms
for developers targeting Linux on ARM64. The [talk](https://pretalx.devconf.info/devconf-cz-2025/talk/P3TEBA/)
at DevConf CZ will focus on the effort to port CentOS Stream to Apple Silicon.
Both sessions will be available online.

## We're chronically online
In addition to our [Mastodon](https://social.treehouse.systems/@AsahiLinux) profile, we now
have Bluesky and LinkedIn accounts. You can follow us at [@asahilinux.org](https://bsky.app/profile/asahilinux.org)
on Bluesky, and at [Asahi Linux](https://www.linkedin.com/company/asahilinux/)
on LinkedIn.

## New distro guidelines
Since the beginning of the project, folks from all walks have worked to support Apple
Silicon in their favourite distros. This immense interest is a gratifying confirmation
that the community values our work.

As part of our effort to encourage distros to take up Apple Silicon support, we have
accommodated all third-party efforts, even allowing distro-specific
documentation on our project's website. Unfortunately, this has led to
an impression that we, as the upstream Asahi Linux developers, are involved
with or otherwise endorse these efforts. This creates both an expectation
of support and an impression that these efforts are representative of the state
of Apple Silicon support, or even the state of the broader AArch64 ecosystem. These expectations are
a significant and growing burden on us, and we need to address it.

We have released [guidelines](https://asahilinux.org/docs/alt/policy/)
outlining our expectations for distros supporting Apple Silicon. These
guidelines target official distro projects wishing to collaborate directly with us. We will
never discourage anyone from adding Apple Silicon support to any distro they choose, but we
cannot offer official support or endorsement to those projects either.

As a result, we are purging all distro-specific documentation and filtering
the list of advertised distros to only those which follow the guidelines.

Our long-term goal remains upstreaming everything such that Apple Silicon does not need
any special treatment or handling.

## Infrastructure ownership
Until recently, most of our infrastructure was under the control of individuals, including
domain names. Over the past month, we have been working on transferring as much of this
as possible away from developers' private accounts and into ownership at the project level.
This ensures that the project is resilient against any one person leaving.
It also makes it easier for project expenses to be accounted for. For example, having
our domain names under the financial ownership of Open Source Collective means that all
domain-related expenses are processed automatically, rather than needing to paid out
by a developer and reimbursed.

## Coming up next...
We have a few items currently pending review on the mailing list, or merged pending
release in Linux 6.16. Of particlar note are drivers for the SMC and SPMI controller.
The SMC is important for system shutdown and reboot, GPIO (required to e.g. power on the WiFi board),
various hardware monitoring sensors, and the RTC. SPMI is a two-wire serial bus similar to I<sup>2</sup>C.
Important peripherals, like the power management controller, are attached via this bus.
Starting with the M3, the USB PD controllers, which negotiate the mode
(e.g. USB3, Display Port, etc.) with the devices attached to the ports and forward
it to the PHY and USB controller, are also attached to SPMI rather than
I<sup>2</sup>C, making the SPMI controller driver essential for supporting
those devices. We hope to have more to share in the next progress report.

As always, we want to thank everyone who supports us on [OpenCollective](https://opencollective.com/asahilinux/)
and [GitHub Sponsors](https://github.com/sponsors/AsahiLinux). None of this would be possible
without your generous support.
