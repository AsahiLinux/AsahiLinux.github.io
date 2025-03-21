+++
date = "2025-03-21T09:00:00+10:00"
draft = false
title = "Progress Report: Linux 6.14"
slug = "progress-report-6-14"
author = "James Calligeros"
+++

As March draws to a close and Linux 6.14 nears release, now is a good time to
provide you all with our first major progress update since [taking the lead on
the project](https://asahilinux.org/2025/02/passing-the-torch/). Going forward,
we hope to keep these updates in sync with upstream kernel releases. We feel
that this is a natural cadence given the focus on upstreaming, with enough time
between posts for noteworthy downstream changes to accumulate.

After getting through all the administrative work required to keep the lights
on after marcan's departure, we've hit the ground running with upstream patch
submission. We held our first board meeting under interesting circumstances,
and we've even managed to sneak a couple of new features in downstream
while you weren't looking. So, without further ado, let's get into it.

## Our first board meeting
We held our inaugural board meeting on 7 March, under challenging conditions
for some. Neal and Davide battled crappy conference hall WiFi while at Southern
California Linux Expo, while I sat in the dark with no power as Tropical Cyclone
Alfred ravaged Southeast Queensland and northern New South Wales. Despite this,
we managed to get through quite a bit. You can find the meeting minutes
[here](https://asahilinux.org/docs/project/board/minutes/20250307).

## Lightening the load
After four years of development, we have racked up a considerable patch
set. Here are some quick-fire stats:

- 90,000+ lines of code added to the Linux kernel as of 6.13, across 1250 patches
- A downstream U-Boot with a number of Apple Silicon and USB stack fixes
- A downstream Mesa - required since our GPU uAPI is not upstream yet
- A downstream virglrenderer - required since our GPU uAPI is not upstream yet
- A downstream Flatpak runtime extension - required since we have a downstream Mesa

This is... less than ideal. Rebasing, testing, and releasing
our forks is a massive drain on our time. This is especially true
of Janne, who has the Sisyphean task of keeping our kernel patch set applying
upstream and porting it to every stable point release. That is hours of work
on an almost weekly basis, especially as the pace of Rust abstraction upstreaming
increases. We can't keep doing this.

We want to bring you M3 and M4 support. We want to bring you Thunderbolt. We want
to bring you DisplayPort Alt Mode. We want to bring you Variable Refresh Rate, and
HDR, and hardware accelerated video playback, and better power management for the
Pro/Max/Ultra Macs. To do that, however, we must start reducing the amount of patches
we're carrying downstream. Most of what we're carrying is stable and has been
for years. So after a month of tireless effort, how are we doing?

We have submitted three new drivers upstream - the Image Signal Processor (ISP)
driver, which is necessary for webcam support, and drivers for the Touchbar's
display controller and input digitiser. These three drivers total to almost
8,000 lines of code, with the ISP driver being 6,000 of those. The good news is
that both Touchbar drivers have already been accepted! Thanks to chaos_princess
for taking on the responsibility of preparing and submitting all three.

Alyssa and Janne have been hard at work tidying up the GPU driver to prepare it for
submission. This has involved some changes to the uAPI, which should slightly
improve performance for some workloads.

Now that Rust for Linux abstractions are starting to be merged at a healthy
pace, we are faced with an emerging challenge. It is rare for
any kernel patch to survive the mailing list without at least a couple of
non-trivial changes, and Rust abstractions are no exception. Every time an
abstraction used by our driver is merged, we must drop our downstream version
and rebase the driver atop the version accepted upstream. This is gruelling, menial,
and unpleasant work, and Janne has our deepest gratitude for volunteering his
time to get through it.

We have also been working to clean up and upstream other parts of the kernel.
In addition to a number of miscellaneous fixes and changes for drivers already
upstreamed such as the NVMe and I<sup>2</sup>C controllers, we have also submitted
changes to the upstream Texas Instruments TAS2764 and TAS2770 speaker amplifier
drivers which extend them to support the Apple-specific variants found in
Apple Silicon Macs. Through this process, we found that the ASoC maintainers
had already been cherry-picking some commits from our development branches!

We're far from done, but we are committed to getting through this vital work.
More patches are being submitted all the time, so watch this space!

## Is this thing on?
By the time you're reading this, we will have enabled microphone support on 
most laptops! This has been a long time coming, with Martin Povišer, Eileen
Yoon, and chaos_princess having reverse engineered and developed a driver
for the Always-On Processor (AOP) in Rust quite some time ago. This is Apple
though. Nothing is ever simple.

Before even getting to play with the mics in Linux, we hit a hurdle.
On certain machines, the Secure Enclave is in the path of the physical mic
data lines. If SEP is unhappy, you don't get mic access. chaos_princess
quickly reverse engineered the SEP endpoint responsible, and wrote a
stub driver to simply toggle the hardware mic switch on. With data now
coming in to ALSA on all machines, we can continue.

In all laptops released so far, Apple use three Pulse Density Modulation
(PDM) mics wired up to an ADC and decimator in the AOP. All three mics are
plumbed directly to userspace on separate channels, with no preamplification.
Having three mics enables Apple to use beamforming, which is a signal processing
technique to greatly enhance the directivity and sensitivity of an array of sensors.
Originally developed for RADAR and military communications, it's now mostly known
for being a marketing dot point for WiFi access points. It's also really, really,
really hard.

Unfortunately, PDM mics are very omnidirectional and very sensitive. We cannot
get by without some kind of beamforming. I initially tried
a very basic delay-and-sum beamformer, which involves no advanced signal
processing. However since the mics are only about 2cm apart and only sample at
48 kHz, we cannot achieve the temporal resolution required to make this
approach work. Signal processing it is, then.

The mysteries of signal processing in the acoustic domain are truly understood by
very few. After consulting the scant available literature on the matter and my high
school maths textbooks, I thought I had absolutely no hope of doing a good job of this.
I put out a call for help on Mastodon, however no one offered to step up. Fine, I'll
do it myself.

After reading a few more papers, I found an approach that looked familiar to me.
A Minimum Variance Distortionless Response beamformer works by taking your complex
sample data in the time domain and using optimisation techniques weighted in a
particular direction relative to a fixed point (usually some mic in the array).
At some point, it clicked that this is really just statistics!

After wrestling with Rust's immature scientific computing libraries for a few weeks,
and even more reading of my high school maths and university statistics textbooks,
the result was [Triforce](https://crates.io/crates/triforce-lv2) - an MVDR beamformer
for the mics found in Apple laptops.

Thanks to the groundwork laid in PipeWire and WirePlumber for speaker support,
wiring up a DSP chain including Triforce for the microphones was really simple.
We just had to update the config files, and let WirePlumber figure out the rest!

Did it work? It's Rust, so it worked on the first try! When properly dialled in,
the mic array is sensitive to signals coming from a "typical" seating position in
front of the laptop, and mostly rejects noise from any other direction. Try it
yourself - play some music out of your phone and move it around your laptop. It
should become suppressed when moved out to the sides or behind the machine.

Once again, a huge thanks to Martin, Eileen, and chaos_princess for their incredible
work reversing AOP and SEP.

## Fedora Asahi Remix 42 Beta
Thanks to Davide's work validating builds, we are pleased to announce that Fedora
Asahi Remix 42 is [now available](https://fedoramagazine.org/announcing-fedora-asahi-remix-42-beta/)
as a beta! The final release is expected in about a month, bringing us closer than
ever to releasing at the same time as Fedora. We hope that in the next cycle
Fedora Asahi Remix 43 will release on the same day as Fedora Linux 43.

## Asahi Linux @ SCaLE
While at [SCaLE](https://www.socallinuxexpo.org/scale/22x), Neal and Davide managed
to set up a demonstration of Asahi Linux running on an M2 Mac mini and an M1 Mac Studio
in the expo hall. Reception was extremely positive, and folks had a blast playing a whole
bunch of Steam games through muvm and FEX! We even managed to get kids involved by firing
up Nidhogg, which proved to be the most popular game by far.

## WoAoA
Some astute users have noticed that ARM64 Windows VMs now work with KVM on
Asahi! This is thanks to Oliver Upton, who submitted a rather large series of
patches upstream to enable Arm PMUv3 emulation on Apple SoCs. WoA requires
PMUv3, and until now we could not emulate it. We quietly cherry-picked those
patches, and they are now incorporated into our latest kernels. Interested
parties can try Windows on ARM on Asahi today!

## What does the dxdiag say about its Feature Level?
It's 12_0! Alyssa recently added sparse binding support to our Vulkan driver,
which was the last piece of the puzzle for Direct3D Feature Level 12_0 via vkd3d-proton. 
Many more D3D12 games now run with the latest Mesa and FEX rootfs.
Our Vulkan driver is not yet as mature as our OpenGL driver though, so performance
and playability vary... but optimisations are coming soon.

## Magic pairing
Apple Silicon Macs have a neat feature whereby Bluetooth devices remain
synchronised across all macOS installs, even recoveryOS. Information required
to pair devices is stored in the machine's NVRAM. macOS checks the NVRAM
for devices that it doesn't know about, and copies that data. Since
they see the same Bluetooth controller in the Mac regardless of
which macOS install is booted, devices will happily connect automatically without
having to go through the pairing process again. This is really useful for
ensuring that Bluetooth keyboards and mice work seamlessly across all environments,
especially in recoveryOS.

To replicate this in Linux, Janne developed [asahi-btsync](https://crates.io/crates/asahi-btsync).
The tool reads the Bluetooth device information out of NVRAM and injects it into
BlueZ's configuration. Up until a few weeks ago this was a manual process which
required running asahi-btsync and connecting to the discovered devices manually.
This is impractical for input devices which are required to be working for these
actions.

As of version 2.4, asahi-btsync integrates with systemd and D-Bus to
fully automate the Bluetooth device synchronisation process. Starting with Fedora
Asahi Remix 42, all Bluetooth LE devices paired with macOS will be available from
your first boot - including the initial Fedora Asahi Remix setup!

Do note however that this does not apply to Bluetooth 5.0 devices, which use a
new authentication and encryption scheme.

## OpenCollective and GitHub Sponsors
When we stood up our [OpenCollective](https://opencollective.com/asahilinux),
none of us really knew what to expect. We had hoped to maybe scrape together
enough support to keep the domains and CDN going, but we dared not hope any harder.

The sheer volume of support and the speed at which it flowed
in left us floored and humbled beyond measure. Your faith in our work means more
than can be expressed in words, and we seriously cannot thank you all enough.

The financial support provided via OpenCollective allows us to continue our work
with confidence. Not only do we have the cash we need to keep the project's
existing infrastructure online, but we have the resources we have always wanted
to ensure the project's viability long into the future.

Until now, we have been mostly purchasing machines out of our own pockets. Given
the nature of what we do, this can mean spending upwards of $10,000 a year on
Macs. Not only is this unsustainable for us on an individual level, it
also presents an enormous barrier to entry for talented folks who may not have
the means to buy such expensive computers. The financial support provided by
OpenCollective backers enables us to signficantly reduce that financial
burden. This has already been invaluable for development work.

The M1 MacBook Air is our most popular Mac by far, with almost 14,000 installations
and counting. This represents almost 30% of the total install base. We couldn't
possibly release microphone support without supporting it. Being able to simply
buy the machine and get the work done without worrying about food for the next
two weeks is the only reason we were able to ship mics in a timely manner.

Buying hardware is not the only game-changing benefit afforded to us by your
financial support. We can now actually build and maintain the CI farm we have
sorely needed for years, which will accelerate development. We can afford email
hosting and other collaborative infrastructure that greatly reduces friction.
We can go to conferences and events without fear of going into financial stress.
All of this is necessary to ensure the long-term sustainability of Asahi as a
project, and none of it would be possible without you.

For those who would prefer to support us via GitHub Sponsors, it's in the works.
We have put in the request with GitHub to link our Sponsors account to our OpenCollective,
however this has not yet been approved. Stay tuned for more.

## What's next?
While this month has been a whirlwind, literally in my case, we've started
settling down into a rhythm that will work nicely in the near-term. Our primary
focus is to continue submitting kernel patches upstream. Reducing our maintenance
burden is the only way we will be able to get back to the fun stuff sustainably.

Neal and Davide will be [presenting](https://events.experiences.redhat.com/widget/redhat/sum25/SessionCatalog2025/session/1731519631980001Xort) at [Red Hat Summit](https://www.redhat.com/en/summit) on 19 May,
demonstrating Fedora Asahi Remix and the upcoming CentOS Hyperscale Asahi Remix
to showcase how Asahi has made it easier than ever for developers to start bringing
their workloads to ARM64 Linux.

We hope to have even more good news for you when Linux 6.15 releases, so stay tuned
for more updates! Once again, we sincerely thank everyone who has supported us so far.
