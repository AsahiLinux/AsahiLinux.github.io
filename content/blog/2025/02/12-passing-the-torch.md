+++
date = "2025-02-13T14:00:00-00:00"
draft = false
title = "Passing the torch on Asahi Linux"
slug = "passing-the-torch"
author = "The Asahi Linux team"
+++

With a heavy heart, we announce the resignation of Asahi Linux founder Hector
Martin (marcan). His statement is on [his
blog](https://marcan.st/2025/02/resigning-as-asahi-linux-project-lead/).  Asahi
Linux brings Linux to Apple Silicon, supporting audio, webcams, graphics
acceleration, and more. As the remaining developers, we are taking this as an
opportunity to build sustainable project governance. No matter how talented an
individual, a large project cannot rest on a single person's shoulders. So
instead of one replacement... we have seven:

* [**Alyssa Rosenzweig**](https://rosenzweig.io), graphics dev.
* [**chaos_princess**](https://social.treehouse.systems/@chaos_princess), kernel dev.
* [**Davide Cavalca**](https://github.com/davide125), Fedora dev.
* [**Neal Gompa**](https://royalgeekworld.com/), Fedora dev.
* [**James Calligeros**](https://social.treehouse.systems/@chadmed), audio dev.
* [**Janne Grunau**](https://social.treehouse.systems/@janne), kernel dev.
* [**Sven Peter**](https://social.treehouse.systems/@sven), kernel dev.

When it comes to project decision-making, we will share equal power in
accordance with our [new governance](/governance). Nobody's contributions last
forever. These governance changes will allow the project to persist as
developers come and go.

Asahi Linux relies primarily on volunteer contributors. Although some
contributors have individual Patreon or GitHub Sponsors accounts, individual
funding streams cannot sustain a team. Going forward, our new fiscal sponsor
Open Source Collective will instead facilitate donations to the project as a
whole.

Our [Open Collective](https://opencollective.com/asahilinux) therefore replaces
marcan's Patreon as the primary funding source for the project. The Patreon
will wind down soon. Four years ago, your Patreon support made this project
possible. As his Patreon is winding down, today we ask for your support to make
the project possible for years to come. Your support will allow us to purchase
hardware and fund developer time. Please consider joining us on [Open
Collective](https://opencollective.com/asahilinux)<!-- or our [GitHub
Sponsors](https://github.com/sponsors/AsahiLinux) --> to continue supporting
the project.

What can you look forward to in 2025?

Our priority is kernel upstreaming. Our downstream Linux tree contains over
1000 patches required for Apple Silicon that are not yet in upstream Linux. The
upstream kernel moves fast, requiring us to constantly rebase our changes on
top of upstream while battling merge conflicts and regressions. Janne, Neal,
and marcan have rebased our tree for years, but it is laborious with so many
patches.  Before adding more, we need to reduce our patch stack to remain
sustainable long-term. We cannot predict how the process will go, but we are
committed to do our part.

The other sustainability issue is testing. We must ensure that every supported
feature works on all supported hardware, with no regressions over time. As we
support more features and hardware, the testing requirements explode.
Unfortunately, manual testing is time-intensive, and bugs *still* slip through.
The solution is continuous integration (CI) that automatically tests Asahi
Linux on many devices.  Like upstreaming, building this infrastructure is not
glamorous, but it will protect the project's long-term health.

Where do the M3 and M4 fit in? Until upstreaming and CI progress, the core team
cannot prioritize new hardware. Nevertheless, some community members are busy
reverse-engineering to prepare for when the foundations are solid.

Of course, there *are* new features coming for M1 and M2 devices. In 2025, we
expect to release...

* **DP alt mode**, required for external monitors over USB-C on laptops without a physical HDMI port.
* **Sparse images** in our [Vulkan driver](/2024/06/vk13-on-the-m1-in-1-month/), enabling DirectX 12. In the mean
  time, you can [enjoy DirectX 11 games](/2024/10/aaa-gaming-on-asahi-linux/) on Asahi Linux.
* **Internal microphones**. External mics already work via the 3.5mm jack, and internal mics are coming soon.

How soon? On select laptops -- just a few days! Microphone support is made
possible by a collaboration between James, chaos_princess, and [Eileen
Yoon](https://github.com/eiln/).  On Apple Silicon, the microphones require
kernel support for multiple hardware blocks, including the Always-On Processor
(AOP) and the Secure Enclave (SEP), as well as userspace support for
[beamforming](https://en.wikipedia.org/wiki/Beamforming) to make sure the audio
sounds great. It's not just samples-in, samples-out... but those three were up
to the challenge.

Today's news is bittersweet. We are grateful to marcan for kicking off this
project and tirelessly working on it these past years. Our community will miss
him. Still, with your support, the project has a bright future to come.
