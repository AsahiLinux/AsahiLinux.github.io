+++
date = "2024-10-10T10:00:00-04:00"
draft = false
title = "AAA gaming on Asahi Linux"
slug = "aaa-gaming-on-asahi-linux"
author = "Alyssa Rosenzweig"
+++

Gaming on Linux on M1 is here! We're thrilled to release our Asahi game playing
toolkit, which integrates our Vulkan 1.3 drivers with x86 emulation and Windows
compatibility. Plus a bonus: conformant OpenCL 3.0.

Asahi Linux now ships the only conformant [OpenGL®](https://www.khronos.org/conformance/adopters/conformant-products/opengl#submission_3470),<!--
[OpenGL® ES](https://www.khronos.org/conformance/adopters/conformant-products/opengles#submission_1045),-->
[OpenCL™](https://www.khronos.org/conformance/adopters/conformant-products/opencl#submission_433), and
[Vulkan®](https://www.khronos.org/conformance/adopters/conformant-products#submission_7910)
drivers for this hardware. As for gaming... while today's release is an alpha, [**Control**](https://store.steampowered.com/app/870780/Control_Ultimate_Edition/) runs well!

<figure><a href="/img/blog/2024/10/Control-small.png"><img src="/img/blog/2024/10/Control-small.avif" alt="Control"></a></figure>

## Installation

First, install [Fedora Asahi Remix](https://asahilinux.org/fedora/). Once
installed, get the latest drivers with <code style="white-space:nowrap">dnf
upgrade \-\-refresh && reboot</code>. Then just <code
style="white-space:nowrap">dnf install steam</code> and play. While all M1/M2-series systems work, most games require 16GB of memory due to emulation overhead.

## The stack

Games are typically x86 Windows binaries rendering with DirectX, while our target is Arm Linux
with Vulkan. We need to handle each difference:

* [FEX](https://fex-emu.com/) emulates x86 on Arm.
* [Wine](https://www.winehq.org/) translates Windows to Linux.
* [DXVK](https://github.com/doitsujin/dxvk) and [vkd3d-proton](https://github.com/HansKristian-Work/vkd3d-proton) translate DirectX to Vulkan.

There's one curveball: page size. Operating systems allocate memory in fixed
size "pages". If an application expects smaller pages than the system uses,
they will break due to insufficient alignment of allocations. That's a problem:
x86 expects 4K pages but Apple systems use 16K pages.

While Linux can't mix page sizes between processes, it *can* virtualize another
Arm Linux kernel with a different page size. So we run games inside a tiny
virtual machine using [muvm](https://github.com/AsahiLinux/muvm), passing through
devices like the GPU and game controllers. The
hardware is happy because the system is 16K, the game is happy because the
virtual machine is 4K, and you're happy because you can play [**Fallout
4**](https://store.steampowered.com/app/377160/Fallout_4/).

<figure><a href="/img/blog/2024/10/Fallout4-small.png"><img src="/img/blog/2024/10/Fallout4-small.avif" alt="Fallout 4"></a></figure>

## Vulkan

The final piece is an adult-level Vulkan driver, since translating DirectX requires Vulkan 1.3
with many extensions. Back in April, I wrote
[Honeykrisp](https://rosenzweig.io/blog/vk13-on-the-m1-in-1-month.html), the
only Vulkan 1.3 driver for Apple hardware. I've since added DXVK support. Let's look at some new features.

### Tessellation

Tessellation enables games like [**The Witcher 3**](https://store.steampowered.com/app/292030/The_Witcher_3_Wild_Hunt/) to generate
geometry. The M1 has hardware tessellation, but it is
too limited for DirectX, Vulkan, or OpenGL. We must instead tessellate with arcane compute shaders, as detailed in [today's talk at XDC2024](https://www.youtube.com/live/pDsksRBLXPk).

<figure><a href="/img/blog/2024/10/Witcher3-small.png"><img src="/img/blog/2024/10/Witcher3-small.avif" alt="The Witcher 3"></a></figure>

### Geometry shaders

Geometry shaders are an older, cruder method to generate geometry.  Like
tessellation, the M1 lacks geometry shader hardware so we emulate with compute.
Is that fast? No, but geometry shaders are slow [even on desktop
GPUs](http://www.joshbarczak.com/blog/?p=667). They don't need to be fast --
just fast enough for games like
[**Ghostrunner**](https://store.steampowered.com/app/1139900/Ghostrunner/).

<figure><a href="/img/blog/2024/10/Ghostrunner-small.png"><img src="/img/blog/2024/10/Ghostrunner-small.avif" alt="Ghostrunner"></a></figure>

### Enhanced robustness

"Robustness" permits an application's shaders to access buffers out-of-bounds
without crashing the hardware. In OpenGL and Vulkan, out-of-bounds loads may
return arbitrary elements, and out-of-bounds stores may corrupt the buffer.
Our OpenGL driver [exploits this
definition](https://rosenzweig.io/blog/conformant-gl46-on-the-m1.html) for
efficient robustness on the M1.

Some games require stronger guarantees.  In DirectX, out-of-bounds loads return zero, and
out-of-bounds stores are ignored. DXVK therefore requires
[`VK_EXT_robustness2`](https://docs.vulkan.org/guide/latest/robustness.html#_vk_ext_robustness2),
a Vulkan extension strengthening robustness.

Like before, we implement robustness with compare-and-select instructions. A
naïve implementation would *compare* a loaded index with the buffer size and
*select* a zero result if out-of-bounds. However, our GPU loads are vector
while arithmetic is scalar. Even if we disabled page faults, we would need up
to four compare-and-selects per load.

```asm
load R, buffer, index * 16
ulesel R[0], index, size, R[0], 0
ulesel R[1], index, size, R[1], 0
ulesel R[2], index, size, R[2], 0
ulesel R[3], index, size, R[3], 0
```

There's a trick: reserve *64 gigabytes* of zeroes using virtual memory voodoo.
Since every 32-bit index multiplied by 16 fits in 64 gigabytes, any index into
this region loads zeroes. For out-of-bounds loads, we simply replace the buffer
address with the reserved address while preserving the index. Replacing a
64-bit address costs just two 32-bit compare-and-selects.

```asm
ulesel buffer.lo, index, size, buffer.lo, RESERVED.lo
ulesel buffer.hi, index, size, buffer.hi, RESERVED.hi
load R, buffer, index * 16
```

Two instructions, not four.

## Next steps

Sparse texturing is next for Honeykrisp, which will unlock more DX12 games. The alpha already runs DX12 games that don't require sparse, like [**Cyberpunk
2077**](https://store.steampowered.com/app/1091500/Cyberpunk_2077/).

<figure><a href="/img/blog/2024/10/Cyberpunk2077-small.png"><img src="/img/blog/2024/10/Cyberpunk2077-small.avif" alt="Cyberpunk 2077"></a></figure>

While many games are playable, newer AAA titles don't hit 60fps *yet*.
Correctness comes first. Performance improves next. Indie games like
[**Hollow Knight**](https://store.steampowered.com/app/367520/Hollow_Knight/) do run full speed.

<figure><a href="/img/blog/2024/10/HollowKnight-small.png"><img src="/img/blog/2024/10/HollowKnight-small.avif" alt="Hollow Knight"></a></figure>

Beyond gaming, we're adding general purpose x86 emulation based on this
stack. For more information, [see the
FAQ](https://docs.fedoraproject.org/en-US/fedora-asahi-remix/x86-support/).

Today's alpha is a taste of what's to come. Not the final form, but
enough to enjoy [**Portal 2**](https://store.steampowered.com/app/620/Portal_2/) while we work towards "1.0".

<figure><a href="/img/blog/2024/10/Portal2-small.png"><img src="/img/blog/2024/10/Portal2-small.avif" alt="Portal 2"></a></figure>

## Acknowledgements

This work has been years in the making with major contributions from...

* [Alyssa Rosenzweig](https://rosenzweig.io)
* [Asahi Lina](https://lina.yt/me)
* [chaos_princess](https://social.treehouse.systems/@chaos_princess)
* [Davide Cavalca](https://github.com/davide125)
* [Dougall Johnson](https://mastodon.social/@dougall)
* [Ella Stanforth](https://ella.gay)
* [Faith Ekstrand](https://www.gfxstrand.net/faith/welcome/)
* [Janne Grunau](https://social.treehouse.systems/@janne)
* [Karol Herbst](https://chaos.social/@karolherbst)
* [marcan](https://social.treehouse.systems/@marcan)
* [Mary Guillemard](https://mary.zone)
* [Neal Gompa](https://neal.gompa.dev/)
* [Sergio López](https://sinrega.org)
* [TellowKrinkle](https://github.com/TellowKrinkle)
* [Teoh Han Hui](https://github.com/teohhanhui)
* [Rob Clark](https://mastodon.gamedev.place/@robclark)
* [Ryan Houdek](https://github.com/sonicadvance1)

... Plus hundreds of developers whose work we build upon, spanning the Linux,
Mesa, Wine, and FEX projects. Today's release is
thanks to the magic of open source.

We hope you enjoy the magic.

Happy gaming.
