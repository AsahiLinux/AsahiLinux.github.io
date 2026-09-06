+++
date = "2026-09-06T10:30:00+10:00"
draft = false
title = "M2: Episode 1"
slug = "m2-episode-1"
author = "James Calligeros"
+++

It's been a while since we did a blog post outside of the progress reports, and today's as good
an occasion for one as any; support for M3 series machines has now been merged into the installer.
In other words, **Asahi Linux now _officially_ supports Macs with an M3 series SoC!**

Linux support for M3 series SoCs and the machines powered by them is now in a state where _almost_
everything supported on the M1 and M2 series machines just works. This includes the webcam, internal
microphones, USB (up to the hardware limit of USB 3 10 Gb/s), hardware
accelerated video decoding **including support for AV1**, WiFi, Bluetooth, and much more! The only major
exceptions remain full DCP support and the GPU, which we will have more news on in the
[coming months](https://indico.freedesktop.org/event/12/contributions/532/). **Do not expect
performant or power-efficient 3D acceleration right now.**

Given that a lot of this work is fresh, support is gated behind the installer's Expert mode.
Users who wish to give Asahi Linux a try on an M3 series machine can do so by running
```sh
curl -L https://alx.sh/ | EXPERT=1 sh
```
in a macOS terminal and following the prompts. Please remember to do a system upgrade (`dnf upgrade --refresh`)
after you have finished installing. We are aiming to drop the Expert requirement
in time for the release of the Fedora Linux 45 beta in a couple of weeks, barring any major
regressions or other showstopping issues.

There are some known limitations to be aware of, however:
- Sleep currently **does not work** due to limitations with the firmware-provided framebuffer.
  This will be addressed once full DCP support is wired up for M3.
- Similarly, the lack of DCP support means that the HDMI port on equipped MacBooks is currently
  disabled.

As always, we would like to thank our generous supporters on both [OpenCollective](https://opencollective.com/AsahiLinux)
and [GitHub Sponsors](https://github.com/sponsors/AsahiLinux). Support for M3 is the culmination
of years of hard work from multiple people, many of whom have only been able to access M3 hardware
because of your support.

We are looking forward to hearing all your feedback, and hope you enjoy using Asahi Linux
on your M3 devices just as much as we have enjoyed bringing it to you. Stay tuned for
more updates and happy computing!
