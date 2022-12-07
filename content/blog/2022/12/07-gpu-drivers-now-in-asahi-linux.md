+++
date = "2022-12-07T14:50:00+09:00"
draft = false
title = "Apple GPU drivers now in Asahi Linux"
slug = "gpu-drivers-now-in-asahi-linux"
author = "Alyssa Rosenzweig & Asahi Lina"
+++

Hello everyone! We're excited to announce our first public Apple Silicon GPU driver release!

We've been working hard over the past two years to bring this new driver to everyone, and we're really proud to finally be here. This is still an alpha driver, but it's already good enough to run a smooth desktop experience and some games.

Read on to find out more about the state of things today, how to install it (it's an opt-in package), and how to report bugs!

![Quake 3 Arena running on an Apple M1](/img/blog/2022/12/quake3.png)

# Status

This release features work-in-progress OpenGL 2.1 and OpenGL ES 2.0 support for all current Apple M-series systems. That's enough for hardware acceleration with desktop environments, like GNOME and KDE. It's also enough for older 3D games, like Quake3 and Neverball. While there's always room for improvement, the driver is fast enough to run all of the above at 60 frames per second at 4K.

Please note: these drivers have not yet passed the OpenGL (ES) conformance tests. There will be bugs!

What's next? Supporting more applications. While OpenGL (ES) 2 suffices for some applications, newer ones (especially games) demand more OpenGL features. OpenGL (ES) 3 brings with it a slew of new features, like multiple render targets, multisampling, and transform feedback. Work on these features is well under way, but they will each take a great deal of additional development effort, and all are needed before OpenGL (ES) 3.0 is available.

What about Vulkan? We're working on it! Although we're only shipping OpenGL right now, we're designing with Vulkan in mind. Most of the work we're putting toward OpenGL will be reused for Vulkan. We estimated that we could ship working OpenGL 2 drivers much sooner than a working Vulkan 1.0 driver, and we wanted to get hardware accelerated desktops into your hands as soon as possible. For the most part, those desktops use OpenGL, so supporting OpenGL first made more sense to us than diving into the Vulkan deep end, only to use Zink to translate OpenGL 2 to Vulkan to run desktops. Plus, there is a large spectrum of OpenGL support, with OpenGL 2.1 containing a fraction of the features of OpenGL 4.6. The same is true for Vulkan: the baseline Vulkan 1.0 profile is roughly equivalent to OpenGL ES 3.1, but applications these days want Vulkan 1.3 with tons of extensions and "optional" features. Zink's "layering" of OpenGL on top of Vulkan isn't magic: it can only expose the OpenGL features that the underlying Vulkan driver has. A baseline Vulkan 1.0 driver isn't even enough to get OpenGL 2.1 on Zink! Zink itself advertises support for OpenGL 4.6, but of course that's only when paired with Vulkan drivers that support the equivalent of OpenGL 4.6... and that gets us back to a tremendous amount of time and effort.

When will OpenGL 3 support be ready? OpenGL 4? Vulkan 1.0? Vulkan 1.3? In community open source projects, it's said that every time somebody asks when a feature will be done, it delays that feature by a month. Well, a lot of people have been asking...

At any rate, for a sneak peek... here is SuperTuxKart's deferred renderer running at full speed, making liberal use of OpenGL ES 3 features like multiple render targets~

![SuperTuxKart's deferred renderer running on an Apple M1](/img/blog/2022/12/supertuxkart-dr.png)

# Anatomy of a GPU driver

Modern GPUs consist of many distinct "layered" parts. There is...

* a memory management unit and an interface to submit memory-mapped work to the hardware
* fixed-function 3D hardware to rasterize triangles, perform depth/stencil testing, and more
* programmable "shader cores" (like little CPUs with bespoke instruction sets) with work dispatched by the fixed-function hardware

This "layered" hardware demands a "layered" graphics driver stack. We need...

* a kernel driver to map memory and submit memory-mapped work
* a userspace driver to translate OpenGL and Vulkan calls into hardware-specific data structures in graphics memory
* a compiler translating shading programming languages like GLSL to the hardware's instruction set

That's a lot of work, calling for a team effort! Fortunately, that layering gives us natural boundaries to divide work among our small team.

* [Alyssa Rosenzweig](https://social.treehouse.systems/@alyssa) is writing the OpenGL driver and compiler.
* [Asahi Lina](https://vt.social/@lina) is writing the kernel driver and helping with OpenGL.
* [Dougall Johnson](https://mastodon.social/@dougall) is reverse-engineering the instruction set with Alyssa.

Meanwhile, [Ella Stanforth](https://tech.lgbt/@ella) is working on a Vulkan driver, reusing the kernel driver, the compiler, and some code shared with the OpenGL driver.

Of course, we couldn't build an OpenGL driver in under two years just ourselves. Thanks to the power of free and open source software, we stand on the shoulders of FOSS giants. The compiler implements a "NIR" backend, where NIR is a powerful intermediate representation, including GLSL to NIR translation. The kernel driver users the "Direct Rendering Manager" (DRM) subsystem of the Linux kernel to minimize boilerplate. Finally, the OpenGL driver implements the "Gallium3D" API inside of [Mesa](https://mesa3d.org/), the home for open source OpenGL and Vulkan drivers. Through Mesa and Gallium3D, we benefit from thirty years of OpenGL driver development, with common code translating OpenGL into the much simpler Gallium3D. Thanks to the incredible engineering of NIR, Mesa, and Gallium3D, our ragtag team of reverse-engineers can focus on what's left: the Apple hardware.

# Installation instructions

To get the new drivers, you need to run the `linux-asahi-edge` kernel and also install the `mesa-asahi-edge` Mesa package.

```
$ sudo pacman -Syu
$ sudo pacman -S linux-asahi-edge mesa-asahi-edge
$ sudo update-grub
```

Since only one version of Mesa can be installed at a time, pacman will prompt you to replace `mesa` with `mesa-asahi-edge`. This is normal!

We also recommend running Wayland instead of Xorg at this point, so if you're using the KDE Plasma environment, make sure to install the Wayland session:

```
$ sudo pacman -S plasma-wayland-session
```

Then reboot, pick the Wayland session at the top of the login screen (SDDM), and enjoy! You might want to adjust the screen scale factor in *System Settings → Display and Monitor* (Plasma Wayland defaults to 100% or 200%, while 150% is often nicer). If you have "Force font DPI" enabled under *Appearance → Fonts*, you should disable that (it is saved separately for Wayland and Xorg, and shouldn't be necessary on Wayland sessions). Log out and back in for these changes to fully apply.

Xorg and Xorg-based desktop environments should work, but there are a few known issues:

* Expect screen tearing (this might be fixed [soon](https://gitlab.freedesktop.org/xorg/xserver/-/merge_requests/1006))
* VSync does not work (some KDE animations will be too fast, and GL apps will not limit their FPS even with VSync enabled). This is a limitation of Xorg on the Apple DCP display controllers, which do not support VBlank interrupts.
* There are still driver bugs triggered by Xorg/KWin. We're looking into this.

The `linux-asahi-edge` kernel can be installed side-by-side with the standard `linux-asahi` package, but both versions should be kept in sync, so make sure to always update your packages together! You can always pick the `linux-asahi` kernel in the GRUB boot menu, which will disable GPU acceleration and the DCP display driver.

When the packages are updated in the future, it's possible that graphical apps will stop starting up after an update until you reboot, or they may fall back to software rendering. This is normal. Until the UAPI is stable, we'll have to break compatibility between Mesa and the kernel every now and then, so you will need to reboot to make things work after updates. In general, if apps *do* keep working with acceleration after any particular Mesa update, then it's probably safe not to reboot, but you should still do it to make sure you're running the latest kernel!

# Reporting bugs

Since the driver is still in development, there are lots of known issues and we're still working hard on improving conformance test results. Please don't open new bugs for random apps not working! It's still the early days and we know there's a lot of work to do. Here's a quick guide of how to report bugs:

* If you find an app that does not start up at all, please don't report it as a bug. Lots of apps won't work because they require a newer GL version than what we support. Please set the `LIBGL_ALWAYS_SOFTWARE=1` environment variable for those apps to fall back to software rendering. If it is a popular app that is part of the Arch Linux ARM repository, you can make a comment on [this issue](https://github.com/AsahiLinux/linux/issues/73) instead, so we can add Mesa quirks to workaround.
* If you run into issues caused by `linux-asahi-edge` unrelated to the GPU, please add a comment to [this issue](https://github.com/AsahiLinux/linux/issues/70). This includes display output issues! (Resolutions, backlight control, display power control, etc.)
* If the GPU locks up and all GPU apps stop working, run `asahi-diagnose` (for example, from an SSH session), open a new bug on [AsahiLinux/linux](https://github.com/AsahiLinux/linux), attach the file generated by that command, and tell us what you were doing that caused the lockup.
* For other GPU issues (rendering glitches, apps that crash after starting up correctly, and things like that), run `asahi-diagnose` and make a comment on [this issue](https://github.com/AsahiLinux/linux/issues/72), attaching the file generated by that command. Don't forget to tell us about your environment!
* In the future, if a driver update causes a regression (rendering problems or crashes for apps that previously worked properly), you can open a bug [directly in the Mesa tracker](https://gitlab.freedesktop.org/asahi/mesa/-/issues).

We hope you enjoy our driver! Remember, things are still moving quickly, so make sure to update your packages regularly to get updates and bug fixes!

