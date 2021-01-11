---
title: Copyright & Reverse Engineering
date:  "2021-01-05T20:00:00+09:00"
draft: false
---

# Copyright policy

Asahi Linux is an open source project, and all contributions must follow the appropriate open source licenses.

These contribution rules are particularly important for code that is to be upstreamed into other projects, to maintain a clean paper trail of the licensing.

## Licensing

Code developed for Asahi Linux itself should be licensed under a permissive license that allows other projects to take advantage of the code without running into license compatibility problems. The specific licenses are subject to being decided on a case-by-case basis, but we will usually use a permissive license such as the MIT license.

Code developed for other open source projects must be licensed under that same project's license, and follow licensing/header/authorship conventions appropriate for that project. However, specific modules developed by Asahi Linux contributors (such as entire drivers or submodules) should be dual-licensed under a permissive license such as MIT, to ensure that they can be ported or reused within other projects.

For original code, we use [SPDX license identifiers](https://spdx.github.io/spdx-spec/appendix-V-using-SPDX-short-identifiers-in-source-files/) to record the license of individual files in a concise way. Files should have header of this form (with whatever license information is appropriate):

```// SPDX-License-Identifier: GPL-2.0+ OR MIT```

No specific authors should be listed in source code files themselves, as this is hard to maintain and unlikely to remain accurate. For top-level and informational places where a copyright statement is needed, such as the MIT license text or "about" style dialogs, the code should be attributed to "The Asahi Linux Contributors".

We do not require contributors to accept any kind of CLA, nor do we require any kind of copyright assignment. You retain all copyright ownership of any code you write. These are merely conventions about how the origin of the code should be documented in version control and files.

## Attribution

Asahi Linux uses Git for managing source code, and the Git history serves as a record of authorship. The Git "Author" field should reflect the primary author of a change - if you commit a change authored by another person, you should ensure they are listed as the author. If a change is authored by multiple people, you should add one or more `Co-Developed-by: Foo Bar <foo@bar.com>` lines at the end of the commit message.

Non-Git releases of the software will be arranged to have an automatically generated authorship file containing a list of all Git contributors.

In order to certify the origin of contributions, all contributors are required to accept the Developer's Certificate of Origin 1.1:

> ## Developer’s Certificate of Origin 1.1
>
> By making a contribution to this project, I certify that:
>
> * The contribution was created in whole or in part by me and I have the right to submit it under the open source license indicated in the file; or
> * The contribution is based upon previous work that, to the best of my knowledge, is covered under an appropriate open source license and I have the right under that license to submit that work with modifications, whether created in whole or in part by me, under the same open source license (unless I am permitted to submit under a different license), as indicated in the file; or
> * The contribution was provided directly to me by some other person who certified (a), (b) or (c) and I have not modified it.
> * I understand and agree that this project and the contribution are public and that a record of the contribution (including all personal information I submit with it, including my sign-off) is maintained indefinitely and may be redistributed consistent with this project or the open source license(s) involved.

To certify this, add a line to the end of your Git commit message as follows:

```
Signed-off-by: Random J Developer <random@developer.example.org>
```

This can be automated by simply using `git commit -s`.

Real names are not required for authorship or sign-off info, although we encourage those contributors who are comfortable using their real name to do so, as some upstream projects are pickier about this. "Real name" is defined as the name you would use or expect to use for official business and in-person interactions, not whatever name any particular government agency thinks you should be using at the present point in time.


# Reverse engineering policy

We are committed to ensuring that all source code and documentation produced by the project is legal and free of copyright violations and other legal problems. In order to ensure this, we have a policy that all contributors must follow, particularly if they are doing certain kinds of reverse engineering.

"Clean-room" reverse engineering is often considered the gold standard to ensure good legal standing for a reverse engineering project. This involves having separate teams, one of which does the reverse engineering and writes documentation, and the other implements that documentation into the final product. This approach is not a legal requirement to ensure that the final product is free from copyright violations, nor does it absolutely guarantee such a result, but it is a fairly strong legal defense should copyright questions arise.

We recognize that a true textbook clean-room approach is not sustainable for most open source projects of this nature. Thus, we aim to ensure that Asahi Linux's code and contributions are effectively equivalent to what a clean-room approach would produce, without mandating the overhead of a true clean-room process.

In order to protect contributors and developers who want to stay away from such subjects, we require that all reverse engineering discussion happen in the #asahi-re and #asahi-gpu (for GPU RE) IRC channels.

## Non-reverse-engineering development

If you are merely looking at existing Asahi Linux code and improving it (without taking or referencing code from anywhere else), you are not reverse engineering anything, and you do not need to worry about anything else. Just make sure to follow the [Copyright policy](#copyright-policy).

## Referencing other open source code

This is generally OK, but you should not copy and paste any actual code unless the license is compatible and the origin of the code is documented.

In particular, be careful of open source dumps from Apple, as they are often licensed under the APSL, which is not GPL compatible. Asahi Linux does not allow any APSL code to be used directly. You can use it as a reference for how things work, and you can take inspiration from register names (since those are effectively hardware documentation; we do not consider mere last-level register names to be copyrightable), but do not copy whole `#define` blocks or include files, other code, or reimplement identical code flows or algorithms. Follow sensible downstream register naming conventions (such as prefixes).

For example, [this](https://github.com/opensource-apple/xnu/blob/master/pexpert/pexpert/arm64/arm64_common.h#L10) register, as documented in XNU include files, should be defined like this when used for Linux:

```
#define SYS_HID0                  sys_reg(3, 0, 15, 0, 0)
```

Or perhaps:

```
#define SYS_APL_HID0              sys_reg(3, 0, 15, 0, 0)
```

Strictly speaking, referencing incompatibly-licensed code can run into similar copyright issues to binary reverse engineering as detailed below; please make sure to read that section to know what not to do.

Internal tools intended only for exploration and not to be released to end-users are allowed to use APSL code, including taking APSL releases directly and modifying them, as long as all licensing requirements are met. For example, taking an APSL release, modifying it to aid in reverse engineering, and using the result back in macOS is fine, and we can host such changes as an Asahi Linux repo, but they cannot become an actual release of the project.

**Be cautious of open source code that works with Apple-specific undocumented file formats, hardware, and protocols**. Such code should, out of an abundance of caution, be treated as if it were incompatibly licensed, like APSL code. The reason is that we cannot guarantee that said code was not the product of reverse engineering that would violate this policy. Therefore, you should not copy and paste or otherwise directly incorporate such code, even if the license is ostensibly compatible. Generally speaking, it is unlikely that directly incorporating code of this nature would offer more than a minor time saving; they are more useful as stand-alone tools and as code-as-documentation. It is okay to use such third-party stand-alone tools during development, or to fork and patch them for exploration, like APSL code - if a copyright problem surfaces, we can easily remove said tools if they have not tainted other project code. Cases where there is significant value to be derived from integrating such code will be reviewed and audited on a case by case basis by project leads.

Put another way: we want to ensure not just license compatibility, but also compatibility with this policy itself. Therefore, we do not allow combining code developed under this policy with code not developed under this policy in cases where a policy violation could have plausibly occurred.

## Hardware exploration

One particularly safe way to reverse engineer without running into copyright issues is to simply probe the hardware to find out what it does. Any knowledge gained this way is safe to use when writing open source code. For example, dumping register areas, twiddling bits, and seeing what other bits change and how the hardware behaves. This is particularly useful to complete knowledge of a piece of hardware that is only partially understood, and can often lead to insight that is not used and thus not present in the vendor's binary code.

## Register and data tracing

When blind probing isn't enough, another effective trick is to run macOS and look at what it does. This can be done by dumping hardware register state before and after an action, or by intercepting data between different components, such as the userspace graphics stack and the kernel graphics driver, and dumping or modifying data.

Register values and command buffers are usually not considered copyrightable, so it is safe to use this approach to obtain the necessary information to write an open source driver. 

Exceptions include cases where code or large blobs of data are uploaded to the hardware. For example, a binary shader obtained by compiling your own GPU shader source code would normally not be copyrightable, but if it contains significant sections of code inserted by the compiler (for example, significant scaffolding or algorithms to implement a particular feature), then that part may be. Use your best judgement in these cases, and do not copy long instruction sequences that do not directly correspond to code you have written in source form.

Similarly, if any firmware blobs are uploaded to the hardware, those cannot be copied as-is, and we must figure out how to approach them on a case-by-case basis.

## Binary disassembly and decompilation

Asahi Linux does not ban binary disassembly and decompilation as a means to obtain knowledge required to write open source implementations. However, this is a dangerous road. We have seen many cases of "open source" code that was developed by direct translation of reverse engineered binary code. This is unacceptable, and would put the entire project and upstream projects at risk.

Since the Asahi Linux project leads (currently marcan) are ultimately responsible for certifying the origin of code upstreamed as part of the project, we need to ensure that any instances of this approach are following a process that does not result in copyright infringement. We have previously seen other projects take contributions that turned out not to be clean, putting those projects into legal doubt - this was often discovered long after the code was contributed, as the origin of the code may not be immediately apparent. For this reason, we do not currently allow contributions of code developed concurrently with binary reverse engineering except from specific trusted contributors. Please contact marcan for details.

All other contributors are required to follow the textbook "clean-room" approach: If you would like to disassemble/decompile components in order to contribute to the project, you must make that decision for a specific component/area with care, and once you do, you are expected to only contribute to that area by writing clean documentation of the hardware, and let other project members implement it.

Especial caution must be applied for the graphics stack: contributors **must not disassemble or decompile userspace graphics driver binaries**, including the proprietary LLVM-based shader compiler and Metal itself, even to produce documentation for clean room Mesa development. The reason for this is two-fold. One, black box data tracing is a highly effective technique that can be relied on for reverse-engineering in isolation, causing disassembly to be needless risk. Two, userspace code tends to be more algorithmic and original in nature, compared to kernel code, rendering these techniques especially risky. Remember, the goal is to reverse-engineer the GPU hardware, not to reverse-engineer the proprietary driver software.

Contributors doing binary reverse engineering are responsible for any legal consequences of their work, including any consequences of the license associated with said code.

**Important**: In order to ensure the legal safety of project members that don't want to involve themselves in this reverse engineering approach, *all* such discussion is allowed **only** in the #asahi-re IRC channel. This will be strictly enforced, and work involving binary RE on other #asahi-* channels will lead to a kick or a ban. Additionally, do not paste any decompiled/disassembled code into the IRC channels. These channels are publicly logged.

## Usage of unreleased materials

Asahi Linux absolutely forbids the usage of any copyrighted materials not available to the public during reverse engineering. This includes any leaked software (in source or binary form), unreleased documentation, non-public releases (such as restricted betas), etc. Project contributors are expected to refrain from acquiring or using any such content. Only materials that are explicitly made available to the public at large may be used. This applies to materials from both Apple and any other third parties.
