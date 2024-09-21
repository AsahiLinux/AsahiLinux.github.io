+++
date = "2024-06-05T12:00:00+09:00"
draft = false
title = "Vulkan 1.3 on the M1 in 1 month"
slug = "vk13-on-the-m1-in-1-month"
author = "Alyssa Rosenzweig"
+++

<style>u{text-decoration-thickness:0.09em;text-decoration-color:skyblue}</style>

Finally, conformant Vulkan for the M1! The new "Honeykrisp" driver is the first
[conformant
Vulkan®](https://www.khronos.org/conformance/adopters/conformant-products/vulkan#submission_780)
for Apple hardware on any operating system, implementing the full 1.3 spec
without "portability" waivers.

Honeykrisp is **not yet released** for end users. We're
continuing to add features, improve performance, and port to more hardware.
[Source
code](https://gitlab.freedesktop.org/alyssa/mesa/-/tree/honeykrisp-20240506-2/src/asahi/vulkan?ref_type=heads)
is available for developers.

<figure><a href="/img/blog/2024/06/holocure.png"><img src="/img/blog/2024/06/holocure.avif" alt="HoloCure running on Honeykrisp ft. DXVK, FEX, and Proton."></a> <figcaption aria-hidden="true"><a href="https://kay-yu.itch.io/holocure">HoloCure</a> running on Honeykrisp ft. DXVK, FEX, and Proton.</figcaption> </figure>

Honeykrisp is not based on prior M1 Vulkan efforts, but rather
[Faith Ekstrand](https://mastodon.gamedev.place/@gfxstrand)'s open source [NVK
driver](https://www.collabora.com/news-and-blog/news-and-events/introducing-nvk.html)
for NVIDIA GPUs. In her words:

> All Vulkan drivers in Mesa trace their lineage to the Intel Vulkan
> driver and started by copying+pasting from it. My hope is that NVK
> will eventually become the driver that everyone copies and pastes from. To
> that end, I'm building NVK with all the best practices we've developed for
> Vulkan drivers over the last 7.5 years and trying to keep the code-base clean
> and well-organized.

Why spend years implementing features from scratch when we can reuse NVK?
There will be friction starting out, given NVIDIA's desktop architecture
differs from the M1's mobile roots. In exchange, we get a modern driver
designed for desktop games.

We'll need to pass a half-million tests ensuring correctness, [submit the
results](https://www.khronos.org/conformance/adopters), and then we'll become
conformant after 30 days of industry review. Starting from NVK and our OpenGL
4.6 driver... can we write a driver passing the Vulkan 1.3 conformance test
suite *faster* than the 30 day review period?

It's unprecedented...

Challenge accepted.

### April 2

It begins with a text.

> _Faith... I think I want to write a Vulkan driver._

Her advice?

> _Just start typing._

There's no copy-pasting yet -- we just add M1 code to NVK and
remove NVIDIA as we go. Since the kernel mediates our access to the hardware, we
begin connecting "NVK" to [Asahi Lina](https://vt.social/@lina)'s kernel
driver using code shared with OpenGL. Then we plug in our shader
compiler and hit the hay.

### April 3

To access resources, GPUs use "descriptors" containing the address, format, and
size of a resource. Vulkan bundles descriptors into "sets" per the application's "descriptor
set layout". When compiling shaders, the driver lowers descriptor accesses to
marry the set layout with the hardware's data structures. As our descriptors
differ from NVIDIA's, our next task is adapting NVK's descriptor set lowering.
We start with a simple but correct approach, deleting far more code than we
add.

### April 4

With working descriptors, we can compile compute shaders. Now we program
the fixed-function hardware to dispatch compute. We first add
bookkeeping to map Vulkan command buffers to lists of M1 "control streams",
then we generate a compute control stream. We copy that code from our OpenGL
driver, translate the GL into Vulkan, and compute works.

That's enough to move on to "copies" of buffers and images. We implement
Vulkan's copies with compute shaders, internally dispatched
with Vulkan commands as if we were the application. The first copy test
passes.

### April 5

Fleshing out yesterday's code, *all* copy tests pass.

### April 6

We're ready to tackle graphics. The novelty is handling graphics state like
depth/stencil. That's straightforward, but there's a *lot*
of state to handle. Faith's code collects all "dynamic state" into a single
structure, which we translate into hardware control words. As usual, we grab
that translation from our OpenGL driver, blend with NVK, and move on.

### April 7

What makes state "dynamic"? Dynamic state can change without
recompiling shaders. By contrast, static state is baked into shader
binaries called "pipelines". If games create all their pipelines
during a loading screen, there is no compiler "stutter" during gameplay. The
idea hasn't quite panned out: many game developers don't know their state
ahead-of-time so cannot create pipelines early. In response, Vulkan has
<u>[made](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_extended_dynamic_state.html)</u>
<u>[ever](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_extended_dynamic_state2.html)</u>
<u>[more](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_extended_dynamic_state3.html)</u>
<u>[state](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_vertex_input_dynamic_state.html)</u>
<u>[dynamic](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_graphics_pipeline_library.html)</u>, punctuated with the
[`EXT_shader_object`](https://www.khronos.org/blog/you-can-use-vulkan-without-pipelines-today)
extension that makes pipelines *optional*.

We want full dynamic state and shader objects. Unfortunately, the M1 bakes
random state into shaders: vertex attributes, fragment outputs, blending, even
linked interpolation qualifiers. Like most of the industry in the 2010s, the
M1's designers bet on pipelines.

Faced with this hardware, a reasonable driver developer would double-down on
pipelines. DXVK would stutter, but we'd pass conformance.

I am not reasonable.

To eliminate stuttering in OpenGL, we make state dynamic with four strategies:

* Conditional code.
* Precompiled variants.
* Indirection.
* Prologs and epilogs.

Wait, what-a-logs?

AMD also bakes state into shaders... with a twist. They divide the
hardware binary into three parts: a *prolog*, the shader, and an *epilog*.
Confining dynamic state to the periphery eliminates shader variants. They
compile prologs and epilogs on the fly, but that's fast and doesn't stutter.
Linking shader parts is a quick concatenation, or long jumps avoid linking
altogether. This strategy works for the M1, too.

For Honeykrisp, let's follow NVK's lead and treat _all_ state as dynamic.
No other Vulkan driver has implemented full dynamic state and shader objects
this early on, but it avoids refactoring later. Today we add the code to build,
compile, and cache prologs and epilogs.

Putting it together, we get a (dynamic) triangle:

[![Classic rainbow triangle](/img/blog/2024/06/hk-triangle.avif)](/img/blog/2024/06/hk-triangle.png)

### April 8

Guided by the list of failing tests, we wire up the little bits missed along
the way, like translating border colours.

```c
/* Translate an American VkBorderColor into a Canadian agx_border_colour */
enum agx_border_colour
translate_border_color(VkBorderColor color)
{
   switch (color) {
   case VK_BORDER_COLOR_INT_TRANSPARENT_BLACK:
      return AGX_BORDER_COLOUR_TRANSPARENT_BLACK;
   ...
   }
}
```

Test results are getting there.

> **Pass**: 149770, **Fail**: 7741, **Crash**: 2396

That's good enough for [vkQuake](https://github.com/Novum/vkQuake).

[![Vulkan port of Quake running on Honeykrisp](/img/blog/2024/06/vkquake.avif)](/img/blog/2024/06/vkquake.png)\

### April 9

Lots of little fixes bring us to a 99.6% pass rate... for Vulkan 1.1. Why stop
there? NVK is 1.3 conformant, so let's claim 1.3 and skip to the finish line.

> **Pass**: 255209, **Fail**: 3818, **Crash**: 599

98.3% pass rate for 1.3 on our 1 week anniversary.

Not bad.

### April 10

SuperTuxKart has a Vulkan renderer.

[![SuperTuxKart rendering with Honeykrisp, showing Pepper (from Pepper and Carrot) riding her broomstick in the STK Enterprise](/img/blog/2024/06/hkr-stk.avif)](/img/blog/2024/06/hkr-stk.png)

### April 11

[Zink](https://docs.mesa3d.org/drivers/zink.html) works too.

[![SuperTuxKart rendering with Zink on Honeykrisp, same scene but with better lighting](/img/blog/2024/06/hkr-stk-zink.avif)](/img/blog/2024/06/hkr-stk-zink.png)

### April 12

I tracked down some fails to a test bug, where an arbitrary verification
threshold was too strict to pass on some devices. I filed a bug report, and it's
[resolved](https://github.com/KhronosGroup/VK-GL-CTS/commit/5fd73c841d775dff1ad52d8340d79dc120d64696)
within a few weeks.

### April 16

The tests for "descriptor indexing" revealed a compiler bug affecting subgroup
shuffles in non-uniform control flow. The M1's shuffle instruction is quirky,
but it's easy to workaround. Fixing that fixes the descriptor indexing tests.

### April 17

A few tests crash inside our register allocator. Their shaders contain a
peculiar construction:

```c
if (condition) {
   while (true) { }
}
```

`condition` is always false, but the compiler doesn't know that.

Infinite loops are nominally invalid since shaders must terminate in finite
time, but this shader is syntactically valid. "All loops contain a break" seems
obvious for a shader, but it's false. It's straightforward to fix register
allocation, but what a doozy.

### April 18

Remember copies? They're slow, and every frame currently requires a copy to get
on screen.

For "zero copy" rendering, we need enough Linux window system integration to
negotiate an efficient surface layout across process boundaries. Linux uses
"modifiers" for this purpose, so we implement the
[`EXT_image_drm_format_modifier`](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_image_drm_format_modifier.html)
extension. And by implement, I mean copy.

Copies to avoid copies.

### April 20

> _"I'd like a 4K x86 Windows Direct3D PC game on a 16K arm64 Linux Vulkan Mac."_
>
> ...
>
> _"Ma'am, this is a Wendy's."_

### April 22

As bug fixing slows down, we step back and check our driver architecture.
Since we treat all state as dynamic, we don't pre-pack control words during
pipeline creation. That adds theoretical CPU overhead.

Is that a problem? After some optimization,
[vkoverhead](https://github.com/zmike/vkoverhead) says we're pushing 100
million draws per second.

I think we're okay.

### April 24

Time to light up YCbCr. If we don't use special YCbCr hardware,
this feature is "software-only". However, it touches a *lot* of code.

It touches so much code that [Mohamed
Ahmed](https://mohamexiety.github.io/posts/final_report/) spent an entire
summer adding it to NVK.

Which means he spent a summer adding it to Honeykrisp.

Thanks, Mohamed ;-)

### April 25

Query copies are next. In Vulkan, the application can query the number of samples rendered,
writing the result into an opaque
"query pool". The result can be copied from the query pool on the CPU or GPU.

For the CPU, the driver maps the pool's internal data structure and copies the
result. This may require nontrivial repacking.

For the GPU, we need to repack in a compute shader. That's harder, because
we can't just run C code on the GPU, right?

...Actually, we can.

A little witchcraft makes GPU query copies as easy as C.

```c
void copy_query(struct params *p, int i) {
  uintptr_t dst = p->dest + i * p->stride;
  int query = p->first + i;

  if (p->available[query] || p->partial) {
    int q = p->index[query];
    write_result(dst, p->_64, p->results[q]);
  }

  ...
}
```

### April 26

The final boss: border colours, hard mode.

Direct3D lets the application choose an arbitrary border colour when
creating a sampler. By contrast, Vulkan only requires three border colours:

* **`(0, 0, 0, 0)`** -- transparent black
* **`(0, 0, 0, 1)`** -- opaque black
* **`(1, 1, 1, 1)`** -- opaque white

We handled these on April 8. Unfortunately, there are two problems.

First, we need custom border colours for Direct3D compatibility.  Both [DXVK](https://github.com/doitsujin/dxvk) and
[vkd3d-proton](https://github.com/HansKristian-Work/vkd3d-proton) require the
[`EXT_custom_border_color`](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_custom_border_color.html)
extension.

Second, there's a subtle problem with our hardware, causing dozens of fails
even without custom border colours. To understand the issue, let's revisit
texture descriptors, which contain a
pixel _format_ and a component reordering _swizzle_.

Some formats are implicitly reordered. Common "BGRA" formats swap red and blue
for [historical
reasons](https://stackoverflow.com/questions/74924790/why-bgra-instead-of-rgba).
The M1 does not directly support these formats. Instead, the driver composes
the swizzle with the format's reordering.  If the application uses a `BARB`
swizzle with a `BGRA` format, the driver uses an `RABR` swizzle with an
`RGBA` format.

There's a catch: swizzles apply to the border colour, but formats do not.  We
need to *undo* the format reordering when programming the border colour for
correct results after the hardware applies the composed swizzle. Our OpenGL
driver implements border colours this way, because it knows the texture format
when creating the sampler. Unfortunately, Vulkan doesn't give us that
information.

Without custom border colour support, we "should" be okay. Swapping red
and blue doesn't change anything if the colour is white or black.

There's an even *subtler* catch. Vulkan mandates support for a
packed 16-bit format with 4-bit components. The M1 supports a similar format...
but with reversed "endianness", swapping red and *alpha*.

That still seems okay. For transparent black (all zero) and opaque white (all
one), swapping components doesn't change the result.

The problem is opaque black: <code style="white-space:nowrap">(0, 0, 0,
1)</code>.  Swapping red and alpha gives <code style="white-space:nowrap">(1,
0, 0, 0)</code>. Transparent red? Uh-oh.

We're stuck. No known hardware configuration implements correct Vulkan
semantics.

Is hope lost?

Do we give up?

A reasonable person would.

I am not reasonable.

Let's jump into the deep end. If we implement custom border colours, opaque
black becomes a special case. But how? The M1's custom border colours entangle
the texture format with the sampler. A reasonable person would skip Direct3D
support.

As you know, I am not reasonable.

Although the hardware is unsuitable, we control software. Whenever a shader
samples a texture, we'll inject code to fix up the border colour. This
emulation is simple, correct, and slow. We'll use dirty driver
tricks to speed it up later. For now, we eat the cost, advertise full custom border
colours, and pass the opaque black tests.

### April 27

All that's left is some last minute bug fixing, and...

> **Pass**: 686930, **Fail**: 0

Success.

### The future

The next task is implementing everything that
[DXVK](https://github.com/doitsujin/dxvk/blob/master/VP_DXVK_requirements.json)
and
[vkd3d-proton](https://github.com/HansKristian-Work/vkd3d-proton/blob/master/VP_D3D12_VKD3D_PROTON_profile.json)
require to layer Direct3D. That includes esoteric extensions like
[transform feedback](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_EXT_transform_feedback.html). Then [Wine](https://www.winehq.org/) and an [open source x86
emulator](https://github.com/FEX-Emu/FEX) will run Windows games on [Asahi
Linux](https://asahilinux.org/).

That's getting ahead of ourselves. In the mean time, enjoy Linux games with
our [conformant OpenGL
4.6](/img/blog/2024/06/blog/conformant-gl46-on-the-m1.html) drivers... and
stay tuned.

<figure><a href="/img/blog/2024/06/babystorm.png"><img src="/img/blog/2024/06/babystorm.avif" alt="Baby Storm running on Honeykrisp ft. DXVK, FEX, and Proton."></a> <figcaption aria-hidden="true"><a href="https://store.steampowered.com/app/2176400/Baby_Storm/">Baby Storm</a> running on Honeykrisp ft. DXVK, FEX, and Proton.</figcaption> </figure>

---
