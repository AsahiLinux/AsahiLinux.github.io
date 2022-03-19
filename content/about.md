---
title: About
date:  "2021-01-05T20:00:00+09:00"
draft: false
---

# About Asahi Linux

Asahi Linux is a project and community with the goal of porting Linux to Apple Silicon Macs, starting with the 2020 M1 Mac Mini, MacBook Air, and MacBook Pro.

Our goal is not just to make Linux run on these machines but to polish it to the point where it can be used as a daily OS. Doing this requires a tremendous amount of work, as Apple Silicon is an entirely undocumented platform. In particular, we will be reverse engineering the Apple GPU architecture and developing an open-source driver for it.

Asahi Linux is developed by a thriving community of free and open source software developers.

## The name

Asahi means "rising sun" in Japanese, and it is also the name of an apple cultivar. 旭りんご (*asahi ringo*) is what we know as the McIntosh Apple, the apple variety that gave the Mac its name.

## The logo

<img src="/img/AsahiLinux_logomark.svg" alt="Asahi Linux logo" width="100">

The Asahi Linux logo and website were designed by [soundflora*](https://soundflora.tokyo). You can find the logo artwork [here](https://github.com/AsahiLinux/artwork/tree/main/logos).

# FAQ

## What devices will be supported?

All Apple M1 Macs are in scope, as well as future generations as development time permits. The first target will be the M1 Mac Mini.

## Is this a Linux distribution?

Asahi Linux is an overall project to develop support for these Macs. We will eventually release a remix of [Arch Linux ARM](https://archlinuxarm.org/), packaged for installation by end-users, as a distribution of the same name. The majority of the work resides in hardware support, drivers, and tools, and it will be upstreamed to the relevant projects. The distribution will be a convenient package for easy installation by end-users and give them access to bleeding-edge versions of the software we develop.

We expect that support will eventually trickle up and back down to other distributions. Advanced users will always be free to use the distribution of their choice and add the necessary patches/software themselves before this happens.

## Does Apple allow this? Don't you need a jailbreak?

Apple allows booting unsigned/custom kernels on Apple Silicon Macs without a jailbreak! This isn't a hack or an omission, but an actual feature that Apple built into these devices. That means that, unlike iOS devices, Apple does not intend to lock down what OS you can use on Macs (though they probably won't help with the development).

## Is this legal?

As long as no code is taken from macOS to build the Linux support, the result is completely legal to distribute and for end-users to use, as it would not be a derivative work of macOS. Please see our [Copyright & Reverse Engineering Policy](/copyright) for more information.

## How will this be released?

All development takes place on our [GitHub](https://github.com/AsahiLinux). All contributions will be written with the intent to upstream them into the respective upstream projects (starting with the Linux kernel) and upstreamed as early as is practical. Code will be dual-licensed as the upstream license (e.g. GPL) and a permissive license (e.g. MIT), to ensure that the work can be reused in other OSes where possible.

## Will this make Apple Silicon Macs a fully open platform?

No, Apple still controls the boot process and, for example, the firmware that runs on the Secure Enclave Processor. However, no modern device is "fully open" - no usable computer exists today with completely open software and hardware (as much as some companies want to market themselves as such). What ends up changing is where you draw the line between closed parts and open parts. The line on Apple Silicon Macs is when the alternate kernel image is booted, while SEP firmware remains closed - which is quite similar to the line on standard PCs, where the UEFI firmware boots the OS loader, while the ME/PSP firmware remains closed. In fact, mainstream x86 platforms are arguably more intrusive because the proprietary UEFI firmware is allowed to steal the main CPU from the OS at any time via SMM interrupts, which is not the case on Apple Silicon Macs. This has real performance/stability implications; it's not just a philosophical issue.

## Who is working on Asahi Linux?

Asahi Linux is a community, and everyone is invited to contribute. If you are interested in contributing, check out our [contribute page](/contribute)! Major contributors are:

* Hector Martin "marcan", the Asahi project lead. marcan is a seasoned reverse engineer and developer with more than 15 years of experience porting Linux and running unofficial software on undocumented and/or closed devices. This is his most ambitious project yet, and he is funding the effort via [community donations and sponsorship](/support). His previous projects include [PS4 Linux](https://github.com/fail0verflow/ps4-linux), a Linux port to the proprietary hardware found on the PS4, capable of full 3D acceleration using OpenGL and Vulkan (radeon/amdgpu drivers); [AsbestOS](https://github.com/marcan/asbestos), a PS3 Linux bootloader for GameOS mode, and associated kernel patches to make Linux work on the PS3 Slim; and numerous contributions to the [Wii Homebrew ecosystem](https://wiibrew.org/), including being part of the team that developed [The Homebrew Channel](https://wiibrew.org/wiki/Homebrew_Channel) and [BootMii](https://wiibrew.org/wiki/BootMii), documenting much of the hardware, and contributing to open homebrew SDK tooling.

* Alyssa Rosenzweig, the Asahi GPU lead. Alyssa is a Linux graphics hacker known for her work on reverse-engineering the Arm Mali GPUs to build the free Panfrost driver. She is an upstream Mesa3D developer, maintaining both the Panfrost and Asahi drivers.

* Dougall Johnson "dougallj", instruction set architecture extraordinaire. Dougall reverse-engineered much of the instruction set of the Apple GPU and has analyzed the timing of the Apple M1's CPU cores to infer microarchitectural details.

* Sven Peter. Sven has worked tirelessly on upstream Linux support for Apple's Device Address Resolution Table (DART) required for USB, PCIe, Ethernet, and Wi-Fi. He also added USB gadget support to m1n1.

* Mark Kettenis, OpenBSD developer. Mark has written m1n1 and U-Boot drivers for the Apple M1 core peripherals, including the bringup needed for PCIe and NVMe (ANS). Mark has also written OpenBSD drivers for the Apple M1 as a parallel effort to the Linux port.
