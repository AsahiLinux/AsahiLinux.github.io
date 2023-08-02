+++
date = "2023-08-02T23:30:00+09:00"
draft = false
title = "Our new flagship distro: Fedora Asahi Remix"
slug = "fedora-asahi-remix"
author = "marcan"
+++

You've all been waiting for it, many of you have guessed, and now, as announced at [Flock To Fedora](https://flocktofedora.org/), it's time to make it official:

**The new Asahi Linux flagship distribution will be Fedora Asahi Remix!**

We're confident that this new flagship will get us much closer to our goal of a polished Linux experience on Apple Silicon, and we hope you will enjoy using it as much as we're enjoying working on it.

We're still working out the kinks and making things even better, so we are not quite ready to call this a release yet. We aim to officially release the Fedora Asahi Remix **by the end of August 2023**. Look forward to many new features, machine support, and more!

## In the Beginning

From the start of the Asahi Linux project, our goal has been to bring full Linux support to Apple Silicon machines, across all distributions. Supporting new hardware like this, especially hardware this special in the relatively young embedded ARM64 desktop Linux space is no easy task, and involves a huge amount of reverse engineering, development, and integration work, spanning all the way from bootloaders to desktop audio servers!

Much of our initial work focused on the kernel and bootloaders, which can be shared between distros. But as we started reaching the point where kernel support was enough for a (bare-bones) usable system, we still had a lot of distro integration work left. Making hardware work out of the box requires a bunch of subtle integration engineering, as well as working together with userspace-level projects to improve them and add the features we need for these systems.

Our goal is for all distros to eventually integrate all this work, so that users can use their choice of distro and be confident that it will work well on their machine. But, in order to kick off this process, we had to prototype what this integration looks like, which meant we had to create our own distro.

And so, the Asahi Linux Arch Linux ARM remix was born. We took Arch Linux ARM, added our own [overlay package repository](https://github.com/AsahiLinux/PKGBUILDs), and packaged all of our integration work there. Notably, this is a fully downstream project: we have no significant involvement with upstream Arch Linux ARM or Arch Linux, and we directly use the Arch Linux ARM package repositories for the core distro. Our overlay just adds integration scripts, bootloader components, extra userspace support packages (for things like audio), and our forked kernel and Mesa packages.

This worked well to bring Asahi Linux out into the world and the hands of eager users, but it was but a step along the way to our ultimate goal. After all, maintaining bespoke downstream distro remixes is a chore, and we can't rely on unofficial third-party support to bring our work to every other distro. We've always had our sights on deeper cooperation with upstream distros to bring Apple Silicon support directly to them as an officially supported platform, and the Arch ARM integration was mainly intended to serve as a reference for this.

It didn't take long for some people to come knocking on our door...

## Fedora Reaches Out

Very soon after Asahi Linux started (well before our Arch ARM-based release), [Neal Gompa](https://fedoraproject.org/wiki/User:Ngompa) joined our IRC channels and we started talking about working towards integrating our work into Fedora. This was the very first offer to officially collaborate with a major upstream distro, and we were very excited! The Fedora Asahi project started in late 2021, and work began in 2022 alongside the Arch ARM release.

Over the following year, we worked closely with the Fedora folks to fully integrate Apple Silicon support into Fedora, including all our custom packages, kernel and mesa forks, and special image packaging requirements, and now we're finally on the final stretch before release.

## Upstream-First

The Fedora Asahi effort is upstream-first, just like all of our kernel and Mesa work. Our bespoke tools, like the [m1n1](https://packages.fedoraproject.org/pkgs/m1n1/m1n1/) low-level bootloader and our [asahi-scripts](https://packages.fedoraproject.org/pkgs/asahi-scripts/asahi-scripts/) tools, are already in upstream Fedora repositories and available directly to all Fedora users (though they won't do much if you install them on a non-Apple machine!). Meanwhile, our hardware enablement package forks are kept in [COPRs](https://copr.fedorainfracloud.org/groups/g/asahi/coprs/) maintained by the [Fedora Asahi SIG](https://fedoraproject.org/wiki/SIGs/Asahi), built and served from Fedora infra.

Collaborating with distro integration experts and using distro infra like this frees us up to continue focusing on what we do best: reverse engineer hardware and develop bespoke drivers and software. But not only that, it also means we can offer an even better experience for Linux on Apple Silicon users! 

Working directly with upstream means not only can we integrate more closely with the core distribution, but we can also get issues in other packages fixed quickly and smoothly. This is particularly important for platforms like desktop ARM64, where we still run into random app and package bugs quite often. ARM64 desktop Linux has been a niche platform (until now!), and with much less testing comes a higher propensity for bugs, so it's very important that we can address these issues quickly. Fedora already has a very solid, fully supported ARM64 port with a large userbase in the server/headless segment, so it is an excellent base to build upon and help improve the state of desktop Linux on ARM64 for everyone.

We're very happy to have this level of collaboration with Fedora, and the Fedora folks have been an absolutely amazing team throughout this whole effort. We want to thank Davide Cavalca, Eric Curtin, Leif Liddy, Neal Gompa, and Michel Alexandre Salim for kicking off the Asahi SIG and making this all possible.

## Racing to the Finish Line

We still have a lot of work to do, including integrating even more packages for new hardware support and more. Adventurous users can [try out the Fedora Asahi Remix today](https://fedora-asahi-remix.org/), but please expect rough spots (or even complete breakage). We're still very much in the process of integrating everything and a bunch of new features are coming, and things are expected to break while we get everything in shape. Please keep that in mind if you choose to try it ahead of time. We ask that reporters and bloggers wait for the official release before evaluating our work.

We hope you enjoy our efforts when the time for our first official Fedora Asahi Remix release comes. You may be wondering what new features are coming, but we'll have to keep that a secret until release time (stuff isn't even integrated yet, you're not going to get a sneak peek even if you install early). Until then, please hang tight and look forward to the release!
