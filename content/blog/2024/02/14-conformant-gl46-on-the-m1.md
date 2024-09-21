+++
date = "2024-02-14T12:00:00+09:00"
draft = false
title = "Conformant OpenGL 4.6 on the M1"
slug = "conformant-gl46-on-the-m1"
author = "Alyssa Rosenzweig"
+++

For years, the M1 has only supported OpenGL 4.1. That changes
today -- with our release of full OpenGL® 4.6 and OpenGL® ES 3.2!
[Install Fedora](https://fedora-asahi-remix.org/) for the latest M1/M2-series
drivers.

Already installed? Just <code style="white-space:nowrap">dnf upgrade \-\-refresh</code>.

Unlike the vendor's non-conformant 4.1 drivers, our [open
source](https://gitlab.freedesktop.org/asahi/mesa) Linux drivers are
**conformant** to the latest OpenGL versions, finally promising broad
compatibility with modern OpenGL workloads, like
[Blender](https://www.blender.org/).

<a href="/img/blog/2024/02/Blender-Wanderer-high.avif"><img title="Screenshot of Blender running on Apple M1 on Fedora Linux 39. The scene is 'Wanderer', depicting a humanoid in a space suit on a rocky terrain, beside a rover with solar panels." src="/img/blog/2024/02/Blender-Wanderer.avif" width="1465" height="993" style="height:auto;background:linear-gradient(180deg,#000 0%,#000 5%, #bdcbd0 5%,#7d5a37);color:rgba(0,0,0,0)"/></a>

Conformant 4.6/3.2 drivers must pass over 100,000 tests to ensure correctness. The
official list of conformant drivers now includes [our OpenGL
4.6](<https://www.khronos.org/conformance/adopters/conformant-products/opengl#submission_347>)
and [ES
3.2](https://www.khronos.org/conformance/adopters/conformant-products/opengles#submission_1045).

While the vendor doesn't yet support graphics standards like modern OpenGL, we do. For this Valentine's Day, we want to profess our love for
interoperable open standards. We want to free users and developers from lock-in, enabling applications to run anywhere the heart wants without special
ports. For that, we need standards conformance. Six months ago, we became the [first
conformant driver for any standard graphics API for the
M1](/blog/first-conformant-m1-gpu-driver.html) with the release of OpenGL ES
3.1 drivers. Today, we've finished OpenGL with the full 4.6... and we're well
on the road to Vulkan.

---

Compared to 4.1, OpenGL 4.6 adds dozens of required features, including:

* Robustness
* SPIR-V
* [Clip control](/blog/asahi-gpu-part-6.html)
* Cull distance
* [Compute shaders](/blog/first-conformant-m1-gpu-driver.html)
* Upgraded transform feedback

Regrettably, the M1 doesn't map well to any graphics standard newer than OpenGL
ES 3.1. While Vulkan makes some of these features optional, the missing features are
required to layer DirectX and OpenGL on top. No existing solution on M1 gets
past the OpenGL 4.1 feature set.

How do we break the 4.1 barrier? Without hardware support, new features need
new tricks. Geometry shaders, tessellation, and transform feedback become
compute shaders. Cull distance becomes a transformed interpolated value. Clip
control becomes a vertex shader epilogue. The list goes on.

For a taste of the challenges we overcame, let's look at **robustness**.

Built for gaming, GPUs traditionally prioritize raw performance over safety.
Invalid application code, like a shader that reads a buffer out-of-bounds,
can trigger undefined behaviour. Drivers exploit that to maximize performance.

For applications like web browsers, that trade-off is undesirable.
Browsers handle untrusted shaders, which they must sanitize to ensure stability
and security. Clicking a malicious link should not crash the browser. While
some sanitization is necessary as graphics APIs are not security barriers,
reducing undefined behaviour in the API can assist "defence in depth".

"Robustness" features can help. Without robustness, out-of-bounds buffer access
in a shader can crash. With robustness, the application can opt for
defined out-of-bounds behaviour, trading some performance for less attack
surface.

All modern cross-vendor APIs include robustness. Many games even
(accidentally?) rely on robustness. Strangely, the vendor's proprietary
API omits buffer robustness. We must do better for conformance, correctness,
and compatibility.

Let's first define the problem. Different APIs have different definitions of
what an out-of-bounds load returns when robustness is enabled:

* Zero (Direct3D, Vulkan with `robustBufferAccess2`)
* Either zero or some data in the buffer (OpenGL, Vulkan with
 `robustBufferAccess`)
* Arbitrary values, but can't crash (OpenGL ES)

OpenGL uses the second definition: return zero or data from the buffer.
One approach is to return the *last* element of the buffer for
out-of-bounds access. Given the buffer size, we can calculate the last index.
Now consider the *minimum* of the index being accessed and the last index. That
equals the index being accessed if it is valid, and some other valid index
otherwise. Loading the minimum index is safe and gives a spec-compliant result.

As an example, a uniform buffer load without robustness might look like:

```asm
load.i32 result, buffer, index
```

Robustness adds a single unsigned minimum (`umin`) instruction:

```asm
umin idx, index, last
load.i32 result, buffer, idx
```

Is the robust version slower? It can be. The difference should be small
percentage-wise, as arithmetic is faster than memory. With thousands
of threads running in parallel, the arithmetic cost may even be hidden by the
load's latency.

There's another trick that speeds up robust uniform buffers. Like other GPUs,
the M1 supports "preambles". The idea is simple: instead of calculating the
same value in every thread, it's faster to calculate once and reuse the result.
The compiler identifies eligible calculations and moves them to a preamble
executed before the main shader. These redundancies are common, so preambles
provide a nice speed-up.

We usually move uniform buffer loads to the preamble when every thread loads
the same index. Since the size of a uniform buffer is fixed, extra robustness
arithmetic is *also* moved to the preamble. The robustness is "free" for the
main shader. For robust storage buffers, the clamping might move to the
preamble even if the load or store cannot.

Armed with robust uniform and storage buffers, let's consider robust "vertex
buffers". In graphics APIs, the application can set vertex buffers with a base
GPU address and a chosen layout of "attributes" within each buffer. Each
attribute has an offset and a format, and the buffer has a "stride" indicating
the number of bytes per vertex. The vertex shader can then read attributes,
implicitly indexing by the vertex. To do so, the shader loads the address:

<img alt="Base plus stride times vertex plus offset" style="display:block;margin:0 auto;max-width:18em" src="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' style='width:31.875ex;height:2.5ex;vertical-align:-.75ex;margin:1px 0' viewBox='0 -778.581 13744.556 1057.161'%3E%3Cg stroke='%23000' stroke-width='0' transform='scale(1 -1)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23a'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b' x='713'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23c' x='1218'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1617'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='2288'/%3E%3Cg transform='translate(3293)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23f'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23g' x='561'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h' x='955'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='1352'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23j' x='1635'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='2196'/%3E%3C/g%3E%3Cg transform='translate(6105)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23k'/%3E%3Cg transform='translate(394)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23l'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='755'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h' x='1204'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23g' x='1601'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1995'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23m' x='2444'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23n' x='3371'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='10092'/%3E%3Cg transform='translate(11097)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23o'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23p' x='783'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23p' x='1094'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23c' x='1405'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1804'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23g' x='2253'/%3E%3C/g%3E%3C/g%3E%3Cdefs%3E%3Cpath id='k' stroke-width='10' d='M94 250q0 69 10 131t23 107 37 88 38 67 42 52 33 34 25 21h17q14 0 14-9 0-3-17-21t-41-53-49-86-42-138-17-193 17-192 41-139 49-86 42-53 17-21q0-9-15-9h-16l-28 24q-94 85-137 212T94 250Z'/%3E%3Cpath id='e' stroke-width='10' d='M56 237v13l14 20h299v150l1 150q10 13 19 13 13 0 20-15V270h298q15-8 15-20t-15-20H409V-68q-8-14-18-14h-4q-12 0-18 14v298H70q-14 7-14 20Z'/%3E%3Cpath id='n' stroke-width='10' d='m60 749 4 1h22l28-24q94-85 137-212t43-264q0-68-10-131T261 12t-37-88-38-67-41-51-32-33-23-19l-4-4H63q-3 0-5 3t-3 9q1 1 11 13Q221-64 221 250T66 725q-10 12-11 13 0 8 5 11Z'/%3E%3Cpath id='a' stroke-width='10' d='M131 622q-7 7-11 9t-16 3-43 3H28v46h318q77 0 113-5t72-27q43-24 68-61t25-78q0-51-41-93t-107-59l-10-3q73-9 129-55t56-115q0-68-51-120T469 3q-13-2-227-3H28v46h33q42 1 51 3t19 12v561Zm380-109q0 47-26 81t-69 42h-45q-20 0-38 1-67 0-82-1t-19-8q-3-4-3-129V374h83l84 1 10 2q4 1 11 3t25 13 32 24 25 39 12 57Zm26-325q0 51-28 94t-79 54l-101 1H229V116q0-59 5-64 6-5 100-5h49q42 0 60 6 43 14 68 51t26 84Z'/%3E%3Cpath id='b' stroke-width='10' d='M137 305h-22l-37 15-15 39q0 35 34 62t121 27q73 0 118-32t60-76q5-14 5-31t1-115v-70q0-48 5-66t21-18q15 0 20 16t5 53v36h40v-39q-1-40-3-47-9-30-35-47T400-6t-47 18-24 42v4l-2-3q-2-3-5-6t-8-9-12-11-15-12-18-11-22-8-26-6-31-3q-60 0-108 31t-48 87q0 21 7 40t27 41 48 37 78 28 110 15h14v22q0 34-6 50-22 71-97 71-18 0-34-1t-25-4-8-3q22-15 22-44 0-25-16-39Zm-11-199q0-31 24-55t59-25q38 0 67 23t39 60q2 7 3 66 0 58-1 58-8 0-21-1t-45-9-58-20-46-37-21-60Z'/%3E%3Cpath id='c' stroke-width='10' d='M295 316q0 40-27 69t-78 29q-36 0-62-13-30-19-30-52-1-5 0-13t16-24 43-25q18-5 44-9t44-9 32-13q17-8 33-20t32-41 17-62q0-62-38-102T198-10h-8q-52 0-96 36l-8-7-9-9Q71 4 65-1L54-11H42q-3 0-9 6v137q0 21 2 25t10 5h9q12 0 16-4t5-12 7-27 19-42q35-51 97-51 97 0 97 78 0 29-18 47-20 24-83 36t-83 23q-36 17-57 46t-21 62q0 39 17 66t43 40 50 18 44 5h11q40 0 70-15l15-8 9 7q10 9 22 17h12q3 0 9-6V310l-6-6h-28q-6 6-6 12Z'/%3E%3Cpath id='d' stroke-width='10' d='M28 218q0 55 20 100t50 73 65 42 66 15q53 0 91-18t58-50 28-64 9-71q0-7-7-14H126v-15q0-148 100-180 20-6 44-6 42 0 72 32 17 17 27 42l10 24q3 3 16 3h3q17 0 17-10 0-4-3-13-19-55-63-87t-99-32q-95 0-158 69T28 218Zm305 57q-11 128-95 136h-2q-8 0-16-1t-25-8-29-21-23-41-16-66v-7h206v8Z'/%3E%3Cpath id='l' stroke-width='10' d='m114 620-4 4-3 3-4 3q-4 3-5 2t-7 2-11 1-13 1-19 1H19v46h9q18-3 124-3 121 0 142 3h11v-46h-21q-61-3-61-17 0-2 90-248t91-246l86 232q85 230 85 239 0 19-21 29t-46 11h-5v46h9q15-3 115-3 91 0 97 3h6v-46h-7q-75 0-96-41 0-1-112-305T401-14q-5-8-19-8h-15q-14 0-19 8-2 2-117 317-117 314-117 317Z'/%3E%3Cpath id='h' stroke-width='10' d='M36 46h14q39 0 47 14v31q0 14 1 31t0 39 0 42v125l-1 23q-3 19-14 25t-45 9H20v23q0 23 2 23l10 1q10 1 28 2t36 2q16 1 35 2t29 3 11 1h3v-69q39 68 97 68h6q45 0 66-22t21-46q0-21-13-36t-38-15q-25 0-37 16t-13 34q0 9 2 16t5 12 3 5q-2 2-23-4-16-8-24-15-47-45-47-179V101q0-12 1-20t0-15v-5q1-2 3-4t5-3 5-3 7-2 7-1 9-1 9 0 10-1 10 0h31V0h-9q-18 3-127 3Q37 3 28 0h-8v46h16Z'/%3E%3Cpath id='g' stroke-width='10' d='M27 422q53 4 82 56t32 122v15h40V431h135v-46H181V241q1-125 1-141t7-32q14-39 49-39 44 0 54 71 1 8 1 46v35h40v-47q0-77-42-117-27-27-70-27-34 0-59 12t-38 31-19 35-7 32q-1 7-1 148v137H18v37h9Z'/%3E%3Cpath id='m' stroke-width='10' d='M201 0q-12 3-99 3-76 0-85-3h-6v46h14q23 1 42 6t29 9 25 17 18 18 21 26 20 28l46 60-58 78q-9 13-19 27t-16 21-11 15-9 12-6 7-7 6-6 3-6 2-8 2q-6 0-36 2H16v46h7q36-2 103-2 93 0 103 2h8v-46q-36-4-36-16 0-2 10-16t28-38 29-41l4-4 25 34q32 41 32 54 0 6-2 11t-5 7-5 4-7 4l-3 1h-5v46h7q15-3 99-3 79 0 85 3h6v-46h-7q-49 0-81-17-17-8-34-27t-65-84l-16-21 62-85q66-90 71-94t17-7q18-4 53-4h17V0h-14q-8 1-20 1t-25 1-25 0-18 1h-37q-26 0-50-2l-23-1h-9v46h3q11 0 22 5t11 12q0 2-40 57l-41 55q-1-1-31-42t-34-45q-4-5-4-14 0-11 7-19t18-9q2 0 2-23V0h-7Z'/%3E%3Cpath id='f' stroke-width='10' d='M55 507q0 83 57 140t131 57h14q85 0 148-63l21 31q5 7 10 15t10 13l3 4h4q3 0 6 1h4q3 0 9-6V462l-6-6h-18q-11 0-13 3t-5 20q-17 126-101 167-37 16-75 16-53 0-86-36t-33-84q0-34 17-62t48-45q10-4 86-23t84-23q57-22 93-75t37-123q0-81-52-146T301-21q-56 0-100 17t-61 31l-18 14q-4-5-15-20T87-7t-9-14q-2-1-10-1h-4q-3 0-9 6v117q0 119 1 121 2 5 20 5h13q6-6 6-13 0-32 10-63t34-61 66-48 100-18q47 0 81 38t34 93q0 43-22 78t-58 48q-56 14-74 19-5 1-27 6t-33 8-32 11-33 18-29 24-27 35q-30 49-30 105Z'/%3E%3Cpath id='i' stroke-width='10' d='M69 609q0 28 18 44t44 16q23-2 40-17t17-43q0-30-17-45t-42-15q-25 0-42 15t-18 45ZM247 0q-15 3-104 3h-37Q80 3 56 1L34 0h-8v46h16q28 0 49 3 9 4 11 11t2 42v191q0 52-2 66t-14 19q-14 7-47 7H30v23q0 23 2 23l10 1q10 1 28 2t36 2 36 2 29 3 11 1h3V62q5-10 12-12t35-4h23V0h-8Z'/%3E%3Cpath id='j' stroke-width='10' d='M376 495v40q0 24 1 33 0 45-10 56t-51 13h-18v23q0 23 2 23l10 1q10 1 29 2t37 2 37 2 30 3 11 1h3V390q0-306 1-309 3-20 14-26t45-9h18V0q-2 0-76-5t-79-6h-7v55l-8-7q-58-48-130-48-77 0-139 61T34 215q0 100 63 163t147 64q75 0 132-49v102Zm-3-153q-45 63-113 63-49 0-87-36-27-28-34-64t-8-94q0-56 7-91t35-61q30-33 78-33 71 0 122 77v239Z'/%3E%3Cpath id='o' stroke-width='10' d='M56 340q0 83 30 154t78 116 106 70 118 25q133 0 233-104t101-260q0-81-29-150T617 75 510 4 388-22 267 3 160 74 85 189 56 340Zm411 307q-41 18-79 18-28 0-57-11t-62-34-56-71-34-110q-5-28-5-85 0-210 103-293 50-41 108-41h6q83 0 146 79 66 89 66 255 0 57-5 85-21 153-131 208Z'/%3E%3Cpath id='p' stroke-width='10' d='M273 0q-18 3-127 3Q43 3 34 0h-8v46h16q28 0 49 3 8 3 12 11 1 2 1 164v161H33v46h71v66l1 67 2 10q19 65 64 94t95 36h9q8 0 14 1 41-3 62-26t21-52q0-23-14-37t-37-14-37 14-14 37q0 20 18 40h-4q-4 1-11 1-28 0-50-21t-34-55q-6-20-7-95v-66h111v-46H185V225q0-162 1-164t3-4 5-3 5-3 7-2 7-1 9-1 9 0 10-1 10 0h31V0h-9Z'/%3E%3C/defs%3E%3C/svg%3E"/>

Some hardware implements robust vertex fetch natively. Other hardware has
bounds-checked buffers to accelerate robust software vertex fetch.
Unfortunately, the M1 has neither. We need to implement vertex fetch with raw
memory loads.

One instruction set feature helps. In addition to a 64-bit base address,
the M1 GPU's memory loads also take an offset in *elements*. The hardware shifts the offset and
adds to the 64-bit base to determine the address to
fetch. Additionally, the M1 has a combined integer multiply-add instruction
`imad`. Together, these features let us implement vertex loads in two
instructions. For example, a 32-bit attribute load looks like:

```asm
imad idx, stride/4, vertex, offset/4
load.i32 result, base, idx
```

The hardware load can perform an additional small shift. Suppose our attribute
is a vector of 4 32-bit values, densely packed into a buffer with no offset. We can load that attribute in one instruction:

```asm
load.v4i32 result, base, vertex << 2
```

...with the hardware calculating the address:

<img alt="Base plus 4 times vertex left shifted 2, which equals Base plus 16 times vertex" style="display:block;margin:0 auto;max-width:15em" src="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' style='width:25.375ex;height:5.75ex;vertical-align:-2.375ex;margin:1px 0' viewBox='0 -1478.581 10898.731 2457.161'%3E%3Cg stroke='%23000' stroke-width='0'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23a' y='-700' transform='matrix(1 0 0 -1 153 0)'/%3E%3Cg transform='matrix(1 0 0 -1 936 -660)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23c' x='713'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1218'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='1617'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23f' x='2288'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23g' x='3293'/%3E%3Cg transform='translate(3965)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h'/%3E%3Cg transform='translate(394)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='755'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23j' x='1204'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23k' x='1601'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='1995'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23l' x='2444'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23m' x='3648'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23n' x='4931'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23o' x='5436'/%3E%3C/g%3E%3C/g%3E%3Cg transform='matrix(1 0 0 -1 936 700)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23c' x='713'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1218'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='1617'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23f' x='2288'/%3E%3Cg transform='translate(3293)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23p'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23q' x='505'/%3E%3C/g%3E%3Cg transform='translate(4470)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h'/%3E%3Cg transform='translate(394)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='755'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23j' x='1204'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23k' x='1601'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='1995'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23l' x='2444'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23o' x='3371'/%3E%3C/g%3E%3C/g%3E%3C/g%3E%3Cdefs%3E%3Cpath id='a' stroke-width='10' d='M56 347q0 13 14 20h637q15-8 15-20 0-11-14-19l-318-1H72q-16 5-16 20Zm0-194q0 15 16 20h636q14-10 14-20 0-13-15-20H70q-14 7-14 20Z'/%3E%3Cpath id='h' stroke-width='10' d='M94 250q0 69 10 131t23 107 37 88 38 67 42 52 33 34 25 21h17q14 0 14-9 0-3-17-21t-41-53-49-86-42-138-17-193 17-192 41-139 49-86 42-53 17-21q0-9-15-9h-16l-28 24q-94 85-137 212T94 250Z'/%3E%3Cpath id='f' stroke-width='10' d='M56 237v13l14 20h299v150l1 150q10 13 19 13 13 0 20-15V270h298q15-8 15-20t-15-20H409V-68q-8-14-18-14h-4q-12 0-18 14v298H70q-14 7-14 20Z'/%3E%3Cpath id='p' stroke-width='10' d='m213 578-13-5q-14-5-40-10t-58-7H83v46h19q47 2 87 15t56 24 28 22q2 3 12 3 9 0 17-6V361l1-300q7-7 12-9t24-4 62-2h26V0h-11q-21 3-159 3-136 0-157-3H88v46h64q16 0 25 1t16 3 8 2 6 5 6 4v517Z'/%3E%3Cpath id='o' stroke-width='10' d='m60 749 4 1h22l28-24q94-85 137-212t43-264q0-68-10-131T261 12t-37-88-38-67-41-51-32-33-23-19l-4-4H63q-3 0-5 3t-3 9q1 1 11 13Q221-64 221 250T66 725q-10 12-11 13 0 8 5 11Z'/%3E%3Cpath id='n' stroke-width='10' d='M109 429q-27 0-43 18t-16 44q0 71 53 123t132 52q91 0 152-56t62-145q0-43-20-82t-48-68-80-74q-36-31-100-92l-59-56 76-1q157 0 167 5 7 2 24 89v3h40v-3q-1-3-13-91T421 3V0H50v31q0 7 6 15t30 35q29 32 50 56 9 10 34 37t34 37 29 33 28 34 23 30 21 32 15 29 13 32 7 30 3 33q0 63-34 109t-97 46q-33 0-58-17t-35-33-10-19q0-1 5-1 18 0 37-14t19-46q0-25-16-42t-45-18Z'/%3E%3Cpath id='b' stroke-width='10' d='M131 622q-7 7-11 9t-16 3-43 3H28v46h318q77 0 113-5t72-27q43-24 68-61t25-78q0-51-41-93t-107-59l-10-3q73-9 129-55t56-115q0-68-51-120T469 3q-13-2-227-3H28v46h33q42 1 51 3t19 12v561Zm380-109q0 47-26 81t-69 42h-45q-20 0-38 1-67 0-82-1t-19-8q-3-4-3-129V374h83l84 1 10 2q4 1 11 3t25 13 32 24 25 39 12 57Zm26-325q0 51-28 94t-79 54l-101 1H229V116q0-59 5-64 6-5 100-5h49q42 0 60 6 43 14 68 51t26 84Z'/%3E%3Cpath id='c' stroke-width='10' d='M137 305h-22l-37 15-15 39q0 35 34 62t121 27q73 0 118-32t60-76q5-14 5-31t1-115v-70q0-48 5-66t21-18q15 0 20 16t5 53v36h40v-39q-1-40-3-47-9-30-35-47T400-6t-47 18-24 42v4l-2-3q-2-3-5-6t-8-9-12-11-15-12-18-11-22-8-26-6-31-3q-60 0-108 31t-48 87q0 21 7 40t27 41 48 37 78 28 110 15h14v22q0 34-6 50-22 71-97 71-18 0-34-1t-25-4-8-3q22-15 22-44 0-25-16-39Zm-11-199q0-31 24-55t59-25q38 0 67 23t39 60q2 7 3 66 0 58-1 58-8 0-21-1t-45-9-58-20-46-37-21-60Z'/%3E%3Cpath id='d' stroke-width='10' d='M295 316q0 40-27 69t-78 29q-36 0-62-13-30-19-30-52-1-5 0-13t16-24 43-25q18-5 44-9t44-9 32-13q17-8 33-20t32-41 17-62q0-62-38-102T198-10h-8q-52 0-96 36l-8-7-9-9Q71 4 65-1L54-11H42q-3 0-9 6v137q0 21 2 25t10 5h9q12 0 16-4t5-12 7-27 19-42q35-51 97-51 97 0 97 78 0 29-18 47-20 24-83 36t-83 23q-36 17-57 46t-21 62q0 39 17 66t43 40 50 18 44 5h11q40 0 70-15l15-8 9 7q10 9 22 17h12q3 0 9-6V310l-6-6h-28q-6 6-6 12Z'/%3E%3Cpath id='e' stroke-width='10' d='M28 218q0 55 20 100t50 73 65 42 66 15q53 0 91-18t58-50 28-64 9-71q0-7-7-14H126v-15q0-148 100-180 20-6 44-6 42 0 72 32 17 17 27 42l10 24q3 3 16 3h3q17 0 17-10 0-4-3-13-19-55-63-87t-99-32q-95 0-158 69T28 218Zm305 57q-11 128-95 136h-2q-8 0-16-1t-25-8-29-21-23-41-16-66v-7h206v8Z'/%3E%3Cpath id='g' stroke-width='10' d='M462 0q-18 3-129 3-116 0-134-3h-9v46h58q7 0 17 2t14 5 7 8q1 2 1 54v50H28v46l151 231q153 232 155 233 2 2 21 2h18l6-6V211h92v-46h-92V66q0-7 6-12 8-7 57-8h29V0h-9ZM293 211v334L74 212l109-1h110Z'/%3E%3Cpath id='i' stroke-width='10' d='m114 620-4 4-3 3-4 3q-4 3-5 2t-7 2-11 1-13 1-19 1H19v46h9q18-3 124-3 121 0 142 3h11v-46h-21q-61-3-61-17 0-2 90-248t91-246l86 232q85 230 85 239 0 19-21 29t-46 11h-5v46h9q15-3 115-3 91 0 97 3h6v-46h-7q-75 0-96-41 0-1-112-305T401-14q-5-8-19-8h-15q-14 0-19 8-2 2-117 317-117 314-117 317Z'/%3E%3Cpath id='j' stroke-width='10' d='M36 46h14q39 0 47 14v31q0 14 1 31t0 39 0 42v125l-1 23q-3 19-14 25t-45 9H20v23q0 23 2 23l10 1q10 1 28 2t36 2q16 1 35 2t29 3 11 1h3v-69q39 68 97 68h6q45 0 66-22t21-46q0-21-13-36t-38-15q-25 0-37 16t-13 34q0 9 2 16t5 12 3 5q-2 2-23-4-16-8-24-15-47-45-47-179V101q0-12 1-20t0-15v-5q1-2 3-4t5-3 5-3 7-2 7-1 9-1 9 0 10-1 10 0h31V0h-9q-18 3-127 3Q37 3 28 0h-8v46h16Z'/%3E%3Cpath id='k' stroke-width='10' d='M27 422q53 4 82 56t32 122v15h40V431h135v-46H181V241q1-125 1-141t7-32q14-39 49-39 44 0 54 71 1 8 1 46v35h40v-47q0-77-42-117-27-27-70-27-34 0-59 12t-38 31-19 35-7 32q-1 7-1 148v137H18v37h9Z'/%3E%3Cpath id='l' stroke-width='10' d='M201 0q-12 3-99 3-76 0-85-3h-6v46h14q23 1 42 6t29 9 25 17 18 18 21 26 20 28l46 60-58 78q-9 13-19 27t-16 21-11 15-9 12-6 7-7 6-6 3-6 2-8 2q-6 0-36 2H16v46h7q36-2 103-2 93 0 103 2h8v-46q-36-4-36-16 0-2 10-16t28-38 29-41l4-4 25 34q32 41 32 54 0 6-2 11t-5 7-5 4-7 4l-3 1h-5v46h7q15-3 99-3 79 0 85 3h6v-46h-7q-49 0-81-17-17-8-34-27t-65-84l-16-21 62-85q66-90 71-94t17-7q18-4 53-4h17V0h-14q-8 1-20 1t-25 1-25 0-18 1h-37q-26 0-50-2l-23-1h-9v46h3q11 0 22 5t11 12q0 2-40 57l-41 55q-1-1-31-42t-34-45q-4-5-4-14 0-11 7-19t18-9q2 0 2-23V0h-7Z'/%3E%3Cpath id='m' stroke-width='10' d='M639-48q0-6-5-12t-15-7h-1q-6 0-82 41Q430 33 329 88 61 235 59 239q-3 4-3 11t3 11q3 5 277 154t279 152l4 1q3-1 6-1 14-5 14-19 0-8-6-14-1-2-259-143L117 250l257-141Q632-32 633-34q6-6 6-14Zm305 0q0-6-5-12t-15-7h-1q-6 0-82 41Q735 33 634 88 366 235 364 239q-3 4-3 11t3 11q3 5 277 154t279 152l4 1q3-1 6-1 14-5 14-19 0-8-6-14-1-2-259-143L422 250l257-141Q937-32 938-34q6-6 6-14Z'/%3E%3Cpath id='q' stroke-width='10' d='M42 313q0 163 81 258t180 95q69 0 99-36t30-80q0-25-14-40t-39-15q-23 0-38 14t-15 39q0 44 47 53-22 22-62 25-71 0-117-60-47-66-47-202l1-4q5 6 8 13 41 60 107 60h4q46 0 81-19 24-14 48-40t39-57q21-49 21-107v-18q0-23-5-43-11-59-64-115T253-22q-28 0-54 8t-56 30-51 59-36 97-14 141Zm215 84q-30 0-52-17t-34-45-17-57-6-62q0-83 12-119t38-58q24-18 53-18 51 0 78 38 13 18 18 45t5 105q0 80-5 107t-18 45q-27 36-72 36Z'/%3E%3C/defs%3E%3C/svg%3E"/>

What about robustness?

We want to implement robustness with a clamp, like we did for uniform
buffers. The problem is that the vertex buffer size is given in bytes, while
our optimized load takes an index in "vertices". A single vertex buffer can
contain multiple attributes with different formats and offsets, so we can't
convert the size in bytes to a size in "vertices".

Let's handle the latter problem. We can rewrite the addressing equation as:

<img alt="Base plus offset, which is the attribute base, plus stride times vertex" style="display:block;margin:0 auto;max-width:18em" src="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' style='width:33.75ex;height:6ex;vertical-align:-4.25ex;margin:1px 0' viewBox='0 -778.581 14532.556 2598.036'%3E%3Cg stroke='%23000' stroke-width='0' transform='scale(1 -1)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23a'/%3E%3Cg transform='translate(394)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23c' x='713'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1218'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='1617'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23f' x='2682'/%3E%3Cg transform='translate(3687)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23g'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h' x='783'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h' x='1094'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1405'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='1804'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='2253'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23j' x='6334'/%3E%3Cg transform='translate(12 -783)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23k' x='19'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23l' transform='matrix(5.77434 0 0 1 512.872 0)'/%3E%3Cg transform='translate(2909)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23m'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23n' x='455'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23l' transform='matrix(5.77434 0 0 1 3848.094 0)'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23o' x='6249'/%3E%3C/g%3E%3Cg transform='matrix(.7071 0 0 .7071 1116 -1687)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23p'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='754'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='1149'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23q' x='1543'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23r' x='1940'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23s' x='2223'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23t' x='2784'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='3345'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='3739'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23u' x='4188'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23s' x='4443'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23c' x='5004'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='5509'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='5908'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23f' x='6950'/%3E%3Cg transform='translate(7955)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23v'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='561'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23q' x='955'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23r' x='1352'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23w' x='1635'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='2196'/%3E%3C/g%3E%3Cg transform='translate(10767)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23a'/%3E%3Cg transform='translate(394)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23x'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='755'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23q' x='1204'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='1601'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='1995'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23y' x='2444'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23j' x='3371'/%3E%3C/g%3E%3C/g%3E%3Cdefs%3E%3Cpath id='a' stroke-width='10' d='M94 250q0 69 10 131t23 107 37 88 38 67 42 52 33 34 25 21h17q14 0 14-9 0-3-17-21t-41-53-49-86-42-138-17-193 17-192 41-139 49-86 42-53 17-21q0-9-15-9h-16l-28 24q-94 85-137 212T94 250Z'/%3E%3Cpath id='f' stroke-width='10' d='M56 237v13l14 20h299v150l1 150q10 13 19 13 13 0 20-15V270h298q15-8 15-20t-15-20H409V-68q-8-14-18-14h-4q-12 0-18 14v298H70q-14 7-14 20Z'/%3E%3Cpath id='j' stroke-width='10' d='m60 749 4 1h22l28-24q94-85 137-212t43-264q0-68-10-131T261 12t-37-88-38-67-41-51-32-33-23-19l-4-4H63q-3 0-5 3t-3 9q1 1 11 13Q221-64 221 250T66 725q-10 12-11 13 0 8 5 11Z'/%3E%3Cpath id='b' stroke-width='10' d='M131 622q-7 7-11 9t-16 3-43 3H28v46h318q77 0 113-5t72-27q43-24 68-61t25-78q0-51-41-93t-107-59l-10-3q73-9 129-55t56-115q0-68-51-120T469 3q-13-2-227-3H28v46h33q42 1 51 3t19 12v561Zm380-109q0 47-26 81t-69 42h-45q-20 0-38 1-67 0-82-1t-19-8q-3-4-3-129V374h83l84 1 10 2q4 1 11 3t25 13 32 24 25 39 12 57Zm26-325q0 51-28 94t-79 54l-101 1H229V116q0-59 5-64 6-5 100-5h49q42 0 60 6 43 14 68 51t26 84Z'/%3E%3Cpath id='c' stroke-width='10' d='M137 305h-22l-37 15-15 39q0 35 34 62t121 27q73 0 118-32t60-76q5-14 5-31t1-115v-70q0-48 5-66t21-18q15 0 20 16t5 53v36h40v-39q-1-40-3-47-9-30-35-47T400-6t-47 18-24 42v4l-2-3q-2-3-5-6t-8-9-12-11-15-12-18-11-22-8-26-6-31-3q-60 0-108 31t-48 87q0 21 7 40t27 41 48 37 78 28 110 15h14v22q0 34-6 50-22 71-97 71-18 0-34-1t-25-4-8-3q22-15 22-44 0-25-16-39Zm-11-199q0-31 24-55t59-25q38 0 67 23t39 60q2 7 3 66 0 58-1 58-8 0-21-1t-45-9-58-20-46-37-21-60Z'/%3E%3Cpath id='d' stroke-width='10' d='M295 316q0 40-27 69t-78 29q-36 0-62-13-30-19-30-52-1-5 0-13t16-24 43-25q18-5 44-9t44-9 32-13q17-8 33-20t32-41 17-62q0-62-38-102T198-10h-8q-52 0-96 36l-8-7-9-9Q71 4 65-1L54-11H42q-3 0-9 6v137q0 21 2 25t10 5h9q12 0 16-4t5-12 7-27 19-42q35-51 97-51 97 0 97 78 0 29-18 47-20 24-83 36t-83 23q-36 17-57 46t-21 62q0 39 17 66t43 40 50 18 44 5h11q40 0 70-15l15-8 9 7q10 9 22 17h12q3 0 9-6V310l-6-6h-28q-6 6-6 12Z'/%3E%3Cpath id='e' stroke-width='10' d='M28 218q0 55 20 100t50 73 65 42 66 15q53 0 91-18t58-50 28-64 9-71q0-7-7-14H126v-15q0-148 100-180 20-6 44-6 42 0 72 32 17 17 27 42l10 24q3 3 16 3h3q17 0 17-10 0-4-3-13-19-55-63-87t-99-32q-95 0-158 69T28 218Zm305 57q-11 128-95 136h-2q-8 0-16-1t-25-8-29-21-23-41-16-66v-7h206v8Z'/%3E%3Cpath id='x' stroke-width='10' d='m114 620-4 4-3 3-4 3q-4 3-5 2t-7 2-11 1-13 1-19 1H19v46h9q18-3 124-3 121 0 142 3h11v-46h-21q-61-3-61-17 0-2 90-248t91-246l86 232q85 230 85 239 0 19-21 29t-46 11h-5v46h9q15-3 115-3 91 0 97 3h6v-46h-7q-75 0-96-41 0-1-112-305T401-14q-5-8-19-8h-15q-14 0-19 8-2 2-117 317-117 314-117 317Z'/%3E%3Cpath id='q' stroke-width='10' d='M36 46h14q39 0 47 14v31q0 14 1 31t0 39 0 42v125l-1 23q-3 19-14 25t-45 9H20v23q0 23 2 23l10 1q10 1 28 2t36 2q16 1 35 2t29 3 11 1h3v-69q39 68 97 68h6q45 0 66-22t21-46q0-21-13-36t-38-15q-25 0-37 16t-13 34q0 9 2 16t5 12 3 5q-2 2-23-4-16-8-24-15-47-45-47-179V101q0-12 1-20t0-15v-5q1-2 3-4t5-3 5-3 7-2 7-1 9-1 9 0 10-1 10 0h31V0h-9q-18 3-127 3Q37 3 28 0h-8v46h16Z'/%3E%3Cpath id='i' stroke-width='10' d='M27 422q53 4 82 56t32 122v15h40V431h135v-46H181V241q1-125 1-141t7-32q14-39 49-39 44 0 54 71 1 8 1 46v35h40v-47q0-77-42-117-27-27-70-27-34 0-59 12t-38 31-19 35-7 32q-1 7-1 148v137H18v37h9Z'/%3E%3Cpath id='y' stroke-width='10' d='M201 0q-12 3-99 3-76 0-85-3h-6v46h14q23 1 42 6t29 9 25 17 18 18 21 26 20 28l46 60-58 78q-9 13-19 27t-16 21-11 15-9 12-6 7-7 6-6 3-6 2-8 2q-6 0-36 2H16v46h7q36-2 103-2 93 0 103 2h8v-46q-36-4-36-16 0-2 10-16t28-38 29-41l4-4 25 34q32 41 32 54 0 6-2 11t-5 7-5 4-7 4l-3 1h-5v46h7q15-3 99-3 79 0 85 3h6v-46h-7q-49 0-81-17-17-8-34-27t-65-84l-16-21 62-85q66-90 71-94t17-7q18-4 53-4h17V0h-14q-8 1-20 1t-25 1-25 0-18 1h-37q-26 0-50-2l-23-1h-9v46h3q11 0 22 5t11 12q0 2-40 57l-41 55q-1-1-31-42t-34-45q-4-5-4-14 0-11 7-19t18-9q2 0 2-23V0h-7Z'/%3E%3Cpath id='v' stroke-width='10' d='M55 507q0 83 57 140t131 57h14q85 0 148-63l21 31q5 7 10 15t10 13l3 4h4q3 0 6 1h4q3 0 9-6V462l-6-6h-18q-11 0-13 3t-5 20q-17 126-101 167-37 16-75 16-53 0-86-36t-33-84q0-34 17-62t48-45q10-4 86-23t84-23q57-22 93-75t37-123q0-81-52-146T301-21q-56 0-100 17t-61 31l-18 14q-4-5-15-20T87-7t-9-14q-2-1-10-1h-4q-3 0-9 6v117q0 119 1 121 2 5 20 5h13q6-6 6-13 0-32 10-63t34-61 66-48 100-18q47 0 81 38t34 93q0 43-22 78t-58 48q-56 14-74 19-5 1-27 6t-33 8-32 11-33 18-29 24-27 35q-30 49-30 105Z'/%3E%3Cpath id='r' stroke-width='10' d='M69 609q0 28 18 44t44 16q23-2 40-17t17-43q0-30-17-45t-42-15q-25 0-42 15t-18 45ZM247 0q-15 3-104 3h-37Q80 3 56 1L34 0h-8v46h16q28 0 49 3 9 4 11 11t2 42v191q0 52-2 66t-14 19q-14 7-47 7H30v23q0 23 2 23l10 1q10 1 28 2t36 2 36 2 29 3 11 1h3V62q5-10 12-12t35-4h23V0h-8Z'/%3E%3Cpath id='w' stroke-width='10' d='M376 495v40q0 24 1 33 0 45-10 56t-51 13h-18v23q0 23 2 23l10 1q10 1 29 2t37 2 37 2 30 3 11 1h3V390q0-306 1-309 3-20 14-26t45-9h18V0q-2 0-76-5t-79-6h-7v55l-8-7q-58-48-130-48-77 0-139 61T34 215q0 100 63 163t147 64q75 0 132-49v102Zm-3-153q-45 63-113 63-49 0-87-36-27-28-34-64t-8-94q0-56 7-91t35-61q30-33 78-33 71 0 122 77v239Z'/%3E%3Cpath id='g' stroke-width='10' d='M56 340q0 83 30 154t78 116 106 70 118 25q133 0 233-104t101-260q0-81-29-150T617 75 510 4 388-22 267 3 160 74 85 189 56 340Zm411 307q-41 18-79 18-28 0-57-11t-62-34-56-71-34-110q-5-28-5-85 0-210 103-293 50-41 108-41h6q83 0 146 79 66 89 66 255 0 57-5 85-21 153-131 208Z'/%3E%3Cpath id='h' stroke-width='10' d='M273 0q-18 3-127 3Q43 3 34 0h-8v46h16q28 0 49 3 8 3 12 11 1 2 1 164v161H33v46h71v66l1 67 2 10q19 65 64 94t95 36h9q8 0 14 1 41-3 62-26t21-52q0-23-14-37t-37-14-37 14-14 37q0 20 18 40h-4q-4 1-11 1-28 0-50-21t-34-55q-6-20-7-95v-66h111v-46H185V225q0-162 1-164t3-4 5-3 5-3 7-2 7-1 9-1 9 0 10-1 10 0h31V0h-9Z'/%3E%3Cpath id='k' stroke-width='10' d='m-24 327 6 6h33q4 0 7-4t5-7 8-14 19-24q61-81 171-122t216-42q13 0 16-3t3-22V28q0-20-3-24t-15-4q-87 0-182 36Q75 118-16 278l-8 14v35Z'/%3E%3Cpath id='o' stroke-width='10' d='M-10 60v35q0 18 3 21t16 4q142 0 241 51t146 113q8 9 16 21t12 19 7 7q2 2 20 2h17l6-6v-35l-8-14Q375 118 190 36 95 0 8 0-5 0-7 3t-3 21v36Z'/%3E%3Cpath id='m' stroke-width='10' d='M-10 60v51q0 7 5 7 4 2 15 2 86 0 180-36Q375 2 466-158l8-14v-35l-6-6h-34q-3 0-6 4t-5 7-9 15-18 24Q331-82 224-41T9 0Q-4 0-7 3t-3 22v35Z'/%3E%3Cpath id='n' stroke-width='10' d='m-18-213-6 6v35l8 14Q75 2 260 84q74 29 155 35h12q9 0 13 1 14 0 17-3t3-19V25q0-18-3-21t-16-4Q308 0 193-55T25-205q-4-6-7-7t-19-1h-17Z'/%3E%3Cpath id='l' stroke-width='10' d='M-10 0v120h420V0H-10Z'/%3E%3Cpath id='p' stroke-width='10' d='M255 0q-15 3-115 3Q48 3 39 0h-7v46h15q72 3 92 42 1 3 53 157t103 308 53 155q3 8 18 8h10q20-1 24-7 2-2 108-319L617 67q7-13 19-16t51-5h30V0h-9q-9 3-127 3-123 0-144-3h-10v46h13q70 0 70 18 0 2-24 74l-24 71H229l-20-59q-20-59-20-65 0-13 20-26t50-13h5V0h-9Zm192 255L345 557 244 256q0-1 101-1h102Z'/%3E%3Cpath id='s' stroke-width='10' d='M307-11q-73 0-139 66l-10-18q-2-3-5-9t-6-11-4-7l-5-9-20-1H98v298q0 301-1 305-3 19-14 25t-45 9H20v23q0 23 2 23l10 1q10 1 29 2t37 2 37 2 30 3 11 1h3V543q0-152 1-152l3 3q3 3 9 7t15 10 21 10 26 10 32 8 37 3q78 0 138-63t61-163q0-101-64-164T307-11ZM182 98q0-1 5-8t9-11 10-12 12-12 15-11 17-9 21-6 24-3q35 0 68 20t49 67q12 35 12 99 0 75-12 111-27 82-112 82-30 0-61-15t-51-43l-6-8V98Z'/%3E%3Cpath id='t' stroke-width='10' d='M383 58q-56-68-127-68h-7q-125 0-144 99-1 7-2 137-1 109-1 122t-6 21q-10 16-60 16H25v23q0 23 2 23l11 1q10 1 29 2t38 2q17 1 37 2t30 3 12 1h3V261q1-184 3-197 3-15 14-24 20-14 60-14 26 0 47 9t32 23 20 32 12 30 4 24v17q0 16 1 40t0 47v67q0 46-10 57t-50 13h-18v46q2 0 76 5t79 6h7V264q0-180 1-183 3-20 14-26t45-9h18V0q-2 0-75-5t-77-6h-7v69Z'/%3E%3C/defs%3E%3C/svg%3E"/>

That is: one buffer with many attributes at different offsets is equivalent to
many buffers with one attribute and no offset. This gives an alternate
perspective on the same data layout. Is this an improvement? It avoids an
addition in the shader, at the cost of passing more data -- addresses are
64-bit while attribute offsets are
[16-bit](https://vulkan.gpuinfo.org/listreports.php?limit=maxVertexInputAttributeOffset&value=4294967295&platform=all0).
More importantly, it lets us translate the vertex buffer size in bytes into a
size in "vertices" for *each* vertex attribute. Instead of clamping the offset,
we clamp the vertex index. We still make full use of the hardware addressing
modes, now with robustness:

```asm
umin idx, vertex, last valid
load.v4i32 result, base, idx << 2
```

We need to calculate the last valid vertex index ahead-of-time for each
attribute. Each attribute has a format with a particular size. Manipulating
the addressing equation, we can calculate the last *byte* accessed in the
buffer (plus 1) relative to the base:

<img alt="Offset plus stride times vertex plus format" style="display:block;margin:0 auto;max-width:18em" src="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' style='width:31.5ex;height:2.5ex;vertical-align:-.75ex;margin:1px 0' viewBox='0 -778.581 13568.556 1057.161'%3E%3Cg stroke='%23000' stroke-width='0' transform='scale(1 -1)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23a'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b' x='783'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b' x='1094'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23c' x='1405'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1804'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='2253'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23f' x='2869'/%3E%3Cg transform='translate(3874)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23g'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='561'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h' x='955'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='1352'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23j' x='1635'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='2196'/%3E%3C/g%3E%3Cg transform='translate(6686)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23k'/%3E%3Cg transform='translate(394)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23l'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='755'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h' x='1204'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='1601'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1995'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23m' x='2444'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23n' x='3371'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23f' x='10673'/%3E%3Cg transform='translate(11678)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23o'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23p' x='658'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='1496'/%3E%3C/g%3E%3C/g%3E%3Cdefs%3E%3Cpath id='k' stroke-width='10' d='M94 250q0 69 10 131t23 107 37 88 38 67 42 52 33 34 25 21h17q14 0 14-9 0-3-17-21t-41-53-49-86-42-138-17-193 17-192 41-139 49-86 42-53 17-21q0-9-15-9h-16l-28 24q-94 85-137 212T94 250Z'/%3E%3Cpath id='f' stroke-width='10' d='M56 237v13l14 20h299v150l1 150q10 13 19 13 13 0 20-15V270h298q15-8 15-20t-15-20H409V-68q-8-14-18-14h-4q-12 0-18 14v298H70q-14 7-14 20Z'/%3E%3Cpath id='n' stroke-width='10' d='m60 749 4 1h22l28-24q94-85 137-212t43-264q0-68-10-131T261 12t-37-88-38-67-41-51-32-33-23-19l-4-4H63q-3 0-5 3t-3 9q1 1 11 13Q221-64 221 250T66 725q-10 12-11 13 0 8 5 11Z'/%3E%3Cpath id='c' stroke-width='10' d='M295 316q0 40-27 69t-78 29q-36 0-62-13-30-19-30-52-1-5 0-13t16-24 43-25q18-5 44-9t44-9 32-13q17-8 33-20t32-41 17-62q0-62-38-102T198-10h-8q-52 0-96 36l-8-7-9-9Q71 4 65-1L54-11H42q-3 0-9 6v137q0 21 2 25t10 5h9q12 0 16-4t5-12 7-27 19-42q35-51 97-51 97 0 97 78 0 29-18 47-20 24-83 36t-83 23q-36 17-57 46t-21 62q0 39 17 66t43 40 50 18 44 5h11q40 0 70-15l15-8 9 7q10 9 22 17h12q3 0 9-6V310l-6-6h-28q-6 6-6 12Z'/%3E%3Cpath id='d' stroke-width='10' d='M28 218q0 55 20 100t50 73 65 42 66 15q53 0 91-18t58-50 28-64 9-71q0-7-7-14H126v-15q0-148 100-180 20-6 44-6 42 0 72 32 17 17 27 42l10 24q3 3 16 3h3q17 0 17-10 0-4-3-13-19-55-63-87t-99-32q-95 0-158 69T28 218Zm305 57q-11 128-95 136h-2q-8 0-16-1t-25-8-29-21-23-41-16-66v-7h206v8Z'/%3E%3Cpath id='l' stroke-width='10' d='m114 620-4 4-3 3-4 3q-4 3-5 2t-7 2-11 1-13 1-19 1H19v46h9q18-3 124-3 121 0 142 3h11v-46h-21q-61-3-61-17 0-2 90-248t91-246l86 232q85 230 85 239 0 19-21 29t-46 11h-5v46h9q15-3 115-3 91 0 97 3h6v-46h-7q-75 0-96-41 0-1-112-305T401-14q-5-8-19-8h-15q-14 0-19 8-2 2-117 317-117 314-117 317Z'/%3E%3Cpath id='h' stroke-width='10' d='M36 46h14q39 0 47 14v31q0 14 1 31t0 39 0 42v125l-1 23q-3 19-14 25t-45 9H20v23q0 23 2 23l10 1q10 1 28 2t36 2q16 1 35 2t29 3 11 1h3v-69q39 68 97 68h6q45 0 66-22t21-46q0-21-13-36t-38-15q-25 0-37 16t-13 34q0 9 2 16t5 12 3 5q-2 2-23-4-16-8-24-15-47-45-47-179V101q0-12 1-20t0-15v-5q1-2 3-4t5-3 5-3 7-2 7-1 9-1 9 0 10-1 10 0h31V0h-9q-18 3-127 3Q37 3 28 0h-8v46h16Z'/%3E%3Cpath id='e' stroke-width='10' d='M27 422q53 4 82 56t32 122v15h40V431h135v-46H181V241q1-125 1-141t7-32q14-39 49-39 44 0 54 71 1 8 1 46v35h40v-47q0-77-42-117-27-27-70-27-34 0-59 12t-38 31-19 35-7 32q-1 7-1 148v137H18v37h9Z'/%3E%3Cpath id='m' stroke-width='10' d='M201 0q-12 3-99 3-76 0-85-3h-6v46h14q23 1 42 6t29 9 25 17 18 18 21 26 20 28l46 60-58 78q-9 13-19 27t-16 21-11 15-9 12-6 7-7 6-6 3-6 2-8 2q-6 0-36 2H16v46h7q36-2 103-2 93 0 103 2h8v-46q-36-4-36-16 0-2 10-16t28-38 29-41l4-4 25 34q32 41 32 54 0 6-2 11t-5 7-5 4-7 4l-3 1h-5v46h7q15-3 99-3 79 0 85 3h6v-46h-7q-49 0-81-17-17-8-34-27t-65-84l-16-21 62-85q66-90 71-94t17-7q18-4 53-4h17V0h-14q-8 1-20 1t-25 1-25 0-18 1h-37q-26 0-50-2l-23-1h-9v46h3q11 0 22 5t11 12q0 2-40 57l-41 55q-1-1-31-42t-34-45q-4-5-4-14 0-11 7-19t18-9q2 0 2-23V0h-7Z'/%3E%3Cpath id='g' stroke-width='10' d='M55 507q0 83 57 140t131 57h14q85 0 148-63l21 31q5 7 10 15t10 13l3 4h4q3 0 6 1h4q3 0 9-6V462l-6-6h-18q-11 0-13 3t-5 20q-17 126-101 167-37 16-75 16-53 0-86-36t-33-84q0-34 17-62t48-45q10-4 86-23t84-23q57-22 93-75t37-123q0-81-52-146T301-21q-56 0-100 17t-61 31l-18 14q-4-5-15-20T87-7t-9-14q-2-1-10-1h-4q-3 0-9 6v117q0 119 1 121 2 5 20 5h13q6-6 6-13 0-32 10-63t34-61 66-48 100-18q47 0 81 38t34 93q0 43-22 78t-58 48q-56 14-74 19-5 1-27 6t-33 8-32 11-33 18-29 24-27 35q-30 49-30 105Z'/%3E%3Cpath id='i' stroke-width='10' d='M69 609q0 28 18 44t44 16q23-2 40-17t17-43q0-30-17-45t-42-15q-25 0-42 15t-18 45ZM247 0q-15 3-104 3h-37Q80 3 56 1L34 0h-8v46h16q28 0 49 3 9 4 11 11t2 42v191q0 52-2 66t-14 19q-14 7-47 7H30v23q0 23 2 23l10 1q10 1 28 2t36 2 36 2 29 3 11 1h3V62q5-10 12-12t35-4h23V0h-8Z'/%3E%3Cpath id='j' stroke-width='10' d='M376 495v40q0 24 1 33 0 45-10 56t-51 13h-18v23q0 23 2 23l10 1q10 1 29 2t37 2 37 2 30 3 11 1h3V390q0-306 1-309 3-20 14-26t45-9h18V0q-2 0-76-5t-79-6h-7v55l-8-7q-58-48-130-48-77 0-139 61T34 215q0 100 63 163t147 64q75 0 132-49v102Zm-3-153q-45 63-113 63-49 0-87-36-27-28-34-64t-8-94q0-56 7-91t35-61q30-33 78-33 71 0 122 77v239Z'/%3E%3Cpath id='a' stroke-width='10' d='M56 340q0 83 30 154t78 116 106 70 118 25q133 0 233-104t101-260q0-81-29-150T617 75 510 4 388-22 267 3 160 74 85 189 56 340Zm411 307q-41 18-79 18-28 0-57-11t-62-34-56-71-34-110q-5-28-5-85 0-210 103-293 50-41 108-41h6q83 0 146 79 66 89 66 255 0 57-5 85-21 153-131 208Z'/%3E%3Cpath id='b' stroke-width='10' d='M273 0q-18 3-127 3Q43 3 34 0h-8v46h16q28 0 49 3 8 3 12 11 1 2 1 164v161H33v46h71v66l1 67 2 10q19 65 64 94t95 36h9q8 0 14 1 41-3 62-26t21-52q0-23-14-37t-37-14-37 14-14 37q0 20 18 40h-4q-4 1-11 1-28 0-50-21t-34-55q-6-20-7-95v-66h111v-46H185V225q0-162 1-164t3-4 5-3 5-3 7-2 7-1 9-1 9 0 10-1 10 0h31V0h-9Z'/%3E%3Cpath id='o' stroke-width='10' d='M128 619q-7 7-11 9t-16 3-43 3H25v46h557v-4q2-6 14-116t14-116v-4h-40v4q-7 49-9 57-6 37-18 62t-27 38-39 21-46 9-57 2h-88q-34 0-42-2t-11-10q-1-2-1-131V363h71q16 0 24 1t22 3 23 6 17 12q18 18 21 74v21h40V200h-40v21q-3 55-21 75-8 7-18 11t-23 6-21 3-24 1-19 0h-52V189l1-128q7-7 12-9t25-4 63-2h27V0h-12q-24 3-166 3Q51 3 36 0H25v46h33q42 1 51 3t19 12v558Z'/%3E%3Cpath id='p' stroke-width='10' d='M41 46h14q39 0 47 14v62q0 17 1 39t0 42v66q0 35-1 59v23q-3 19-14 25t-45 9H25v23q0 23 2 23l10 1q10 1 28 2t37 2q17 1 36 2t29 3 11 1h3v-40q0-38 1-38t5 5 12 15 19 18 29 19 38 16q20 5 51 5 15 0 28-2t23-6 19-8 15-9 11-11 9-11 7-11 4-10 3-8l2-5 3 4 6 8q3 4 9 11t13 13 15 13 20 12 23 10 26 7 31 3q126 0 137-113 1-7 1-139v-86q0-38 2-45t11-10q21-3 49-3h16V0h-8l-23 1q-24 1-51 1t-38 1Q596 3 587 0h-8v46h16q61 0 61 16 1 2 1 138-1 135-2 143-6 28-20 42t-24 17-26 2q-45 0-79-34-27-27-34-55t-8-83V108q0-30 1-40t3-13 9-6q21-3 49-3h16V0h-8l-24 1q-23 1-50 1t-38 1Q319 3 310 0h-8v46h16q61 0 61 16 1 2 1 138-1 135-2 143-6 28-20 42t-24 17-26 2q-45 0-79-34-27-27-34-55t-8-83V108q0-30 1-40t3-13 9-6q21-3 49-3h16V0h-8l-23 1q-24 1-51 1t-38 1Q42 3 33 0h-8v46h16Z'/%3E%3C/defs%3E%3C/svg%3E"/>

The load is valid when that value is bounded by the buffer size in bytes. We
solve the integer inequality as:

<img alt="Vertex less than or equal to the floor of size minus offset minus format divided by stride" style="display:block;margin:0 auto;max-width:18em" src="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' style='width:34.5ex;height:5.75ex;vertical-align:-2.375ex;margin:1px 0' viewBox='0 -1478.081 14863.222 2456.161'%3E%3Cg stroke='%23000' stroke-width='0' transform='scale(1 -1)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23a'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b' x='755'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23c' x='1204'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1601'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b' x='1995'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23e' x='2444'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23f' x='3254'/%3E%3Cg transform='translate(4315)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23g' x='277' y='-1'/%3E%3Cpath stroke='none' d='M985 220h8853v60H985z'/%3E%3Cg transform='translate(1045 676)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='561'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23j' x='844'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b' x='1293'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23k' x='1964'/%3E%3Cg transform='translate(2969)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23l'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23m' x='783'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23m' x='1094'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23n' x='1405'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b' x='1804'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='2253'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23k' x='5838'/%3E%3Cg transform='translate(6843)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23o'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23p' x='658'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='1496'/%3E%3C/g%3E%3C/g%3E%3Cg transform='translate(4089 -690)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23h'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23d' x='561'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23c' x='955'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23i' x='1352'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23q' x='1635'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23b' x='2196'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23r' x='9959' y='-1'/%3E%3C/g%3E%3C/g%3E%3Cdefs%3E%3Cpath id='n' stroke-width='10' d='M295 316q0 40-27 69t-78 29q-36 0-62-13-30-19-30-52-1-5 0-13t16-24 43-25q18-5 44-9t44-9 32-13q17-8 33-20t32-41 17-62q0-62-38-102T198-10h-8q-52 0-96 36l-8-7-9-9Q71 4 65-1L54-11H42q-3 0-9 6v137q0 21 2 25t10 5h9q12 0 16-4t5-12 7-27 19-42q35-51 97-51 97 0 97 78 0 29-18 47-20 24-83 36t-83 23q-36 17-57 46t-21 62q0 39 17 66t43 40 50 18 44 5h11q40 0 70-15l15-8 9 7q10 9 22 17h12q3 0 9-6V310l-6-6h-28q-6 6-6 12Z'/%3E%3Cpath id='b' stroke-width='10' d='M28 218q0 55 20 100t50 73 65 42 66 15q53 0 91-18t58-50 28-64 9-71q0-7-7-14H126v-15q0-148 100-180 20-6 44-6 42 0 72 32 17 17 27 42l10 24q3 3 16 3h3q17 0 17-10 0-4-3-13-19-55-63-87t-99-32q-95 0-158 69T28 218Zm305 57q-11 128-95 136h-2q-8 0-16-1t-25-8-29-21-23-41-16-66v-7h206v8Z'/%3E%3Cpath id='a' stroke-width='10' d='m114 620-4 4-3 3-4 3q-4 3-5 2t-7 2-11 1-13 1-19 1H19v46h9q18-3 124-3 121 0 142 3h11v-46h-21q-61-3-61-17 0-2 90-248t91-246l86 232q85 230 85 239 0 19-21 29t-46 11h-5v46h9q15-3 115-3 91 0 97 3h6v-46h-7q-75 0-96-41 0-1-112-305T401-14q-5-8-19-8h-15q-14 0-19 8-2 2-117 317-117 314-117 317Z'/%3E%3Cpath id='c' stroke-width='10' d='M36 46h14q39 0 47 14v31q0 14 1 31t0 39 0 42v125l-1 23q-3 19-14 25t-45 9H20v23q0 23 2 23l10 1q10 1 28 2t36 2q16 1 35 2t29 3 11 1h3v-69q39 68 97 68h6q45 0 66-22t21-46q0-21-13-36t-38-15q-25 0-37 16t-13 34q0 9 2 16t5 12 3 5q-2 2-23-4-16-8-24-15-47-45-47-179V101q0-12 1-20t0-15v-5q1-2 3-4t5-3 5-3 7-2 7-1 9-1 9 0 10-1 10 0h31V0h-9q-18 3-127 3Q37 3 28 0h-8v46h16Z'/%3E%3Cpath id='d' stroke-width='10' d='M27 422q53 4 82 56t32 122v15h40V431h135v-46H181V241q1-125 1-141t7-32q14-39 49-39 44 0 54 71 1 8 1 46v35h40v-47q0-77-42-117-27-27-70-27-34 0-59 12t-38 31-19 35-7 32q-1 7-1 148v137H18v37h9Z'/%3E%3Cpath id='e' stroke-width='10' d='M201 0q-12 3-99 3-76 0-85-3h-6v46h14q23 1 42 6t29 9 25 17 18 18 21 26 20 28l46 60-58 78q-9 13-19 27t-16 21-11 15-9 12-6 7-7 6-6 3-6 2-8 2q-6 0-36 2H16v46h7q36-2 103-2 93 0 103 2h8v-46q-36-4-36-16 0-2 10-16t28-38 29-41l4-4 25 34q32 41 32 54 0 6-2 11t-5 7-5 4-7 4l-3 1h-5v46h7q15-3 99-3 79 0 85 3h6v-46h-7q-49 0-81-17-17-8-34-27t-65-84l-16-21 62-85q66-90 71-94t17-7q18-4 53-4h17V0h-14q-8 1-20 1t-25 1-25 0-18 1h-37q-26 0-50-2l-23-1h-9v46h3q11 0 22 5t11 12q0 2-40 57l-41 55q-1-1-31-42t-34-45q-4-5-4-14 0-11 7-19t18-9q2 0 2-23V0h-7Z'/%3E%3Cpath id='h' stroke-width='10' d='M55 507q0 83 57 140t131 57h14q85 0 148-63l21 31q5 7 10 15t10 13l3 4h4q3 0 6 1h4q3 0 9-6V462l-6-6h-18q-11 0-13 3t-5 20q-17 126-101 167-37 16-75 16-53 0-86-36t-33-84q0-34 17-62t48-45q10-4 86-23t84-23q57-22 93-75t37-123q0-81-52-146T301-21q-56 0-100 17t-61 31l-18 14q-4-5-15-20T87-7t-9-14q-2-1-10-1h-4q-3 0-9 6v117q0 119 1 121 2 5 20 5h13q6-6 6-13 0-32 10-63t34-61 66-48 100-18q47 0 81 38t34 93q0 43-22 78t-58 48q-56 14-74 19-5 1-27 6t-33 8-32 11-33 18-29 24-27 35q-30 49-30 105Z'/%3E%3Cpath id='i' stroke-width='10' d='M69 609q0 28 18 44t44 16q23-2 40-17t17-43q0-30-17-45t-42-15q-25 0-42 15t-18 45ZM247 0q-15 3-104 3h-37Q80 3 56 1L34 0h-8v46h16q28 0 49 3 9 4 11 11t2 42v191q0 52-2 66t-14 19q-14 7-47 7H30v23q0 23 2 23l10 1q10 1 28 2t36 2 36 2 29 3 11 1h3V62q5-10 12-12t35-4h23V0h-8Z'/%3E%3Cpath id='q' stroke-width='10' d='M376 495v40q0 24 1 33 0 45-10 56t-51 13h-18v23q0 23 2 23l10 1q10 1 29 2t37 2 37 2 30 3 11 1h3V390q0-306 1-309 3-20 14-26t45-9h18V0q-2 0-76-5t-79-6h-7v55l-8-7q-58-48-130-48-77 0-139 61T34 215q0 100 63 163t147 64q75 0 132-49v102Zm-3-153q-45 63-113 63-49 0-87-36-27-28-34-64t-8-94q0-56 7-91t35-61q30-33 78-33 71 0 122 77v239Z'/%3E%3Cpath id='l' stroke-width='10' d='M56 340q0 83 30 154t78 116 106 70 118 25q133 0 233-104t101-260q0-81-29-150T617 75 510 4 388-22 267 3 160 74 85 189 56 340Zm411 307q-41 18-79 18-28 0-57-11t-62-34-56-71-34-110q-5-28-5-85 0-210 103-293 50-41 108-41h6q83 0 146 79 66 89 66 255 0 57-5 85-21 153-131 208Z'/%3E%3Cpath id='m' stroke-width='10' d='M273 0q-18 3-127 3Q43 3 34 0h-8v46h16q28 0 49 3 8 3 12 11 1 2 1 164v161H33v46h71v66l1 67 2 10q19 65 64 94t95 36h9q8 0 14 1 41-3 62-26t21-52q0-23-14-37t-37-14-37 14-14 37q0 20 18 40h-4q-4 1-11 1-28 0-50-21t-34-55q-6-20-7-95v-66h111v-46H185V225q0-162 1-164t3-4 5-3 5-3 7-2 7-1 9-1 9 0 10-1 10 0h31V0h-9Z'/%3E%3Cpath id='o' stroke-width='10' d='M128 619q-7 7-11 9t-16 3-43 3H25v46h557v-4q2-6 14-116t14-116v-4h-40v4q-7 49-9 57-6 37-18 62t-27 38-39 21-46 9-57 2h-88q-34 0-42-2t-11-10q-1-2-1-131V363h71q16 0 24 1t22 3 23 6 17 12q18 18 21 74v21h40V200h-40v21q-3 55-21 75-8 7-18 11t-23 6-21 3-24 1-19 0h-52V189l1-128q7-7 12-9t25-4 63-2h27V0h-12q-24 3-166 3Q51 3 36 0H25v46h33q42 1 51 3t19 12v558Z'/%3E%3Cpath id='p' stroke-width='10' d='M41 46h14q39 0 47 14v62q0 17 1 39t0 42v66q0 35-1 59v23q-3 19-14 25t-45 9H25v23q0 23 2 23l10 1q10 1 28 2t37 2q17 1 36 2t29 3 11 1h3v-40q0-38 1-38t5 5 12 15 19 18 29 19 38 16q20 5 51 5 15 0 28-2t23-6 19-8 15-9 11-11 9-11 7-11 4-10 3-8l2-5 3 4 6 8q3 4 9 11t13 13 15 13 20 12 23 10 26 7 31 3q126 0 137-113 1-7 1-139v-86q0-38 2-45t11-10q21-3 49-3h16V0h-8l-23 1q-24 1-51 1t-38 1Q596 3 587 0h-8v46h16q61 0 61 16 1 2 1 138-1 135-2 143-6 28-20 42t-24 17-26 2q-45 0-79-34-27-27-34-55t-8-83V108q0-30 1-40t3-13 9-6q21-3 49-3h16V0h-8l-24 1q-23 1-50 1t-38 1Q319 3 310 0h-8v46h16q61 0 61 16 1 2 1 138-1 135-2 143-6 28-20 42t-24 17-26 2q-45 0-79-34-27-27-34-55t-8-83V108q0-30 1-40t3-13 9-6q21-3 49-3h16V0h-8l-23 1q-24 1-51 1t-38 1Q42 3 33 0h-8v46h16Z'/%3E%3Cpath id='f' stroke-width='10' d='M674 636q8 0 14-6t6-15-7-14q-1-1-270-129L151 346l248-118Q687 92 691 87q3-6 3-11 0-18-18-20h-6L382 192Q92 329 90 331q-7 5-7 17 1 11 13 17 8 4 286 135t283 134q4 2 9 2ZM84-118q0 10 15 20h579q16-6 16-20 0-12-15-20H98q-14 7-14 20Z'/%3E%3Cpath id='j' stroke-width='10' d='M42 263q2 7 6 82t5 78v8h340q6-6 6-16 0-12-1-13l-17-24q-17-23-50-69t-66-89L134 41l48-1h24q48 0 77 6t48 31q21 28 28 108l2 16q0 1 20 1h20v-6q0-1-8-93t-9-97V0H209L34 1l-3 2q-3 5-3 14 0 13 1 14t131 179 134 184h-58q-67-1-84-6-25-6-39-21-24-23-31-103v-9H42v8Z'/%3E%3Cpath id='k' stroke-width='10' d='M84 237v13l14 20h581q15-8 15-20t-15-20H98q-14 7-14 20Z'/%3E%3Cpath id='g' stroke-width='10' d='M246-949v2399h62V-887h263v-62H246Z'/%3E%3Cpath id='r' stroke-width='10' d='M274-887v2337h62V-949H11v62h263Z'/%3E%3C/defs%3E%3C/svg%3E"/>

The driver calculates the right-hand side and passes it into the shader.

One last problem: what if a buffer is too small to load *anything*? Clamping
won't save us -- the code would clamp to a negative index. In that case,
the attribute is entirely invalid, so we swap the application's buffer for a
small buffer of zeroes. Since we gave each attribute its own base address,
this determination is per-attribute. Then clamping the index to zero 
correctly loads zeroes.

Putting it together, a little driver math gives us robust buffers at the
cost of one `umin` instruction.

---

In addition to buffer robustness, we need image robustness. Like its buffer
counterpart, image robustness requires that out-of-bounds image loads return
zero. That formalizes a guarantee that reasonable hardware already makes.

...But it would be no fun if our hardware was reasonable.

Running the conformance tests for image robustness, there is a single
test failure affecting "mipmapping".

For background, mipmapped images contain multiple "levels of detail". The base
level is the original image; each successive level is the previous level
downscaled. When rendering, the hardware selects the level closest to matching
the on-screen size, improving efficiency and visual quality.

With robustness, the specifications all agree that image loads return...

* Zero if the X- or Y-coordinate is out-of-bounds
* Zero if the level is out-of-bounds

Meanwhile, image loads on the M1 GPU return...

* Zero if the X- or Y-coordinate is out-of-bounds
* Values from the last level if the level is out-of-bounds

Uh-oh. Rather than returning zero for out-of-bounds levels, the hardware
clamps the level and returns nonzero values. It's a mystery why.
The vendor does not document their hardware publicly, forcing us to
rely on reverse engineering to build drivers. Without documentation,
we don't know if this behaviour is intentional or a hardware bug. Either way,
we need a workaround to pass conformance.

The obvious workaround is to never load from an invalid level:

```glsl
if (level <= levels) {
    return imageLoad(x, y, level);
} else {
    return 0;
}
```

That involves branching, which is inefficient. Loading an out-of-bounds level
doesn't crash, so we can speculatively load and then use a compare-and-select
operation instead of branching:

```glsl
vec4 data = imageLoad(x, y, level);

return (level <= levels) ? data : 0;
```

This workaround is okay, but it could be improved. While the M1 GPU has combined
compare-and-select instructions, the instruction set is *scalar*. Each thread
processes one value at a time, not a vector of multiple values. However, image
loads return a vector of four components (red, green, blue, alpha). While the
pseudo-code looks efficient, the resulting assembly is not:

```asm
image_load R, x, y, level
ulesel R[0], level, levels, R[0], 0
ulesel R[1], level, levels, R[1], 0
ulesel R[2], level, levels, R[2], 0
ulesel R[3], level, levels, R[3], 0
```

Fortunately, the vendor driver has a trick. We know the hardware returns zero
if either X or Y is out-of-bounds, so we can *force* a zero output by *setting*
X or Y out-of-bounds. As the maximum image size is 16384 pixels wide, any X
greater than 16384 is out-of-bounds. That justifies an alternate workaround:

```glsl
bool valid = (level <= levels);
int x_ = valid ? x : 20000;

return imageLoad(x_, y, level);
```

Why is this better? We only change a single scalar, not a whole vector,
compiling to compact scalar assembly:

```asm
ulesel x_, level, levels, x, #20000
image_load R, x_, y, level
```

If we preload the constant to a uniform register, the workaround is a single
instruction. That's optimal -- and it passes conformance.

---

_Blender ["Wanderer"](https://download.blender.org/demo/eevee/wanderer/wanderer.blend) demo by [Daniel Bystedt](https://www.artstation.com/dbystedt), licensed CC BY-SA._
