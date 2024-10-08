+++
date = "2023-08-22T23:30:00+09:00"
draft = false
title = "The first conformant M1 GPU driver"
slug = "first-conformant-m1-gpu-driver"
author = "Alyssa Rosenzweig"
+++

Conformant OpenGL® ES 3.1 drivers are now available for M1- and M2-family GPUs.
That means the drivers are compatible with any OpenGL ES 3.1 application.
Interested? [Just install Linux!](https://fedora-asahi-remix.org/)

For existing [Asahi Linux](https://asahilinux.org/) users,
upgrade your system with  <code style="white-space:nowrap;">dnf
upgrade</code> (Fedora) or <code style="white-space:nowrap;">pacman -Syu</code>
(Arch) for the latest drivers.

Our reverse-engineered, free and [open source graphics
drivers](https://gitlab.freedesktop.org/asahi/mesa) are the world's ***only***
conformant OpenGL ES 3.1 implementation for M1- and M2-family graphics
hardware. That means our driver passed tens of thousands of tests to
demonstrate correctness and is now recognized by the industry.

To become conformant, an "implementation" must pass the official conformance
test suite, designed to verify every feature in the specification. The test
results are submitted to Khronos, the standards body. After a [30-day review
period](https://www.khronos.org/conformance/adopters/), if no issues are found,
the implementation becomes conformant. The Khronos website lists all conformant
implementations, including our drivers for the
[M1](https://www.khronos.org/conformance/adopters/conformant-products/opengles#submission_1007),
[M1
Pro/Max/Ultra](https://www.khronos.org/conformance/adopters/conformant-products/opengles#submission_1014),
[M2](https://www.khronos.org/conformance/adopters/conformant-products/opengles#submission_1016),
and [M2
Pro/Max](https://www.khronos.org/conformance/adopters/conformant-products/opengles#submission_1017).

Today's milestone isn't just about OpenGL ES. We're releasing the first
conformant implementation of *any* graphics standard for the M1. And we don't
plan to stop here ;-)

[![Teaser of the "Vulkan instancing" demo running on Asahi Linux](/img/blog/2023/08/vkinstancing2.webp)](/img/blog/2023/08/vkinstancing.webp)

Unlike ours, the manufacturer's M1 drivers are unfortunately not conformant for _any_
standard graphics API, whether Vulkan or OpenGL or OpenGL ES. That means that
there is no guarantee that applications using the standards will work on your M1/M2 (if you're
not running Linux). This isn't just a theoretical issue. Consider Vulkan.
The third-party [MoltenVK](https://github.com/KhronosGroup/MoltenVK)
layers a subset of Vulkan on top of the proprietary drivers. However, those drivers
lack key functionality, breaking valid Vulkan applications. That hinders
developers and users alike, if they haven't yet switched their M1/M2 computers
to Linux.

Why did *we* pursue standards conformance when the manufacturer did not? Above
all, our commitment to quality. We want our users to know that they can depend
on our Linux drivers. We want standard software to run without M1-specific
hacks or porting. We want to set the right example for the ecosystem: the way forward is
implementing open standards, conformant to the specifications, without
compromises for "portability". We are not satisfied with proprietary
drivers, proprietary APIs, and refusal to implement standards. The rest of the
industry knows that progress comes from cross-vendor collaboration. We know it,
too. Achieving conformance is a win for our community, for open source, and for
open graphics.

Of course, [Asahi Lina](https://vt.social/@lina/) and I are two individuals
with minimal funding. It's a little awkward that we beat the big corporation...

It's not too late though. They should follow our lead!

---

OpenGL ES 3.1 updates the experimental [OpenGL ES 3.0 and OpenGL
3.1](/img/blog/2024/02/blog/opengl3-on-asahi-linux.html) we shipped in
June. Notably, ES 3.1 adds compute shaders, typically used to accelerate
general computations within graphics applications. For example, a 3D game could
run its physics simulations in a compute shader. The simulation results can
then be used for rendering, eliminating stalls that would otherwise be required
to synchronize the GPU with a CPU physics simulation. That lets the game run
faster.

Let's zoom in on one new feature: atomics on images. Older versions of
OpenGL ES allowed an application to read an image in order to display it on screen.
ES 3.1 allows the application to *write* to the image, typically from a
compute shader. This new feature enables flexible image processing algorithms, which
previously needed to fit into the fixed-function 3D pipeline. However, GPUs
are massively parallel, running thousands of threads at the same time. If two
threads write to the same location, there is a conflict: depending which thread
runs first, the result will be different. We have a race condition.

"Atomic" access to memory provides a solution to race conditions. With atomics,
special hardware in the memory subsystem guarantees consistent, well-defined
results for select operations, regardless of the order of the threads. Modern
graphics hardware supports various atomic operations, like addition,
serving as building blocks to complex parallel algorithms.

Can we put these two features together to write to an image atomically?

Yes. A ubiquitous OpenGL ES
[extension](https://registry.khronos.org/OpenGL/extensions/OES/OES_shader_image_atomic.txt),
required for ES 3.2, adds atomics operating on pixels in an image. For
example, a compute shader could atomically increment the value at pixel (10,
20).

Other GPUs have dedicated instructions to perform atomics on an images, making
the driver implementation straightforward. For us, the story is more
complicated. The M1 lacks hardware instructions for image atomics, even though
it has non-image atomics and non-atomic images. We need to reframe the
problem.

The idea is simple: to perform an atomic on a pixel, we instead calculate
the address of the pixel in memory and perform a regular atomic on that
address. Since the hardware supports regular atomics, our task is "just"
calculating the pixel's address.

If the image were laid out linearly in memory, this would be straightforward:
multiply the Y-coordinate by the number of bytes per row ("stride"), multiply
the X-coordinate by the number of bytes per pixel, and add.  That gives the
pixel's offset in bytes relative to the first pixel of the image. To get the
final address, we add that offset to the address of the first pixel.

<img alt="Address of (X, Y) equals Address of (0, 0) + Y times Stride + X times Bytes Per Pixel" src="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' viewBox='0 -776.7226410624452 27924.66666666667 1053.4452821248904' style='width: 64.861ex; height: 2.5ex; vertical-align: -0.694ex; margin: 1px 0px;'%3E%3Cg stroke='black' fill='black' stroke-width='0' transform='matrix(1 0 0 -1 0 0)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-41'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-64' x='755' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-64' x='1316' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-72' x='1877' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-65' x='2274' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-73' x='2723' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-73' x='3122' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-28' x='3521' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMATHI-58' x='3915' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-2C' x='4772' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMATHI-59' x='5221' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-29' x='5989' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-3D' x='6661' y='0'/%3E%3Cg transform='translate(7722,0)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-41'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-64' x='755' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-64' x='1316' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-72' x='1877' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-65' x='2274' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-73' x='2723' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-73' x='3122' y='0'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-28' x='11243' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-30' x='11637' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-2C' x='12142' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-30' x='12591' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-29' x='13096' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-2B' x='13713' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMATHI-59' x='14718' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-22C5' x='15708' y='0'/%3E%3Cg transform='translate(16213,0)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-53'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-74' x='561' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-72' x='955' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-69' x='1352' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-64' x='1635' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-65' x='2196' y='0'/%3E%3C/g%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-2B' x='19081' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMATHI-58' x='20086' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-22C5' x='21165' y='0'/%3E%3Cg transform='translate(21670,0)'%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-42'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-79' x='713' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-74' x='1246' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-65' x='1640' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-73' x='2089' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-50' x='2488' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-65' x='3174' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-72' x='3623' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-50' x='4020' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-69' x='4706' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-78' x='4989' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-65' x='5522' y='0'/%3E%3Cuse xmlns:xlink='http://www.w3.org/1999/xlink' xlink:href='%23MJMAIN-6C' x='5971' y='0'/%3E%3C/g%3E%3C/g%3E%3Cdefs id='MathJax_SVG_glyphs'%3E%3Cpath id='MJSZ2-2211' stroke-width='10' d='M60 948Q63 950 665 950H1267L1325 815Q1384 677 1388 669H1348L1341 683Q1320 724 1285 761Q1235 809 1174 838T1033 881T882 898T699 902H574H543H251L259 891Q722 258 724 252Q725 250 724 246Q721 243 460 -56L196 -356Q196 -357 407 -357Q459 -357 548 -357T676 -358Q812 -358 896 -353T1063 -332T1204 -283T1307 -196Q1328 -170 1348 -124H1388Q1388 -125 1381 -145T1356 -210T1325 -294L1267 -449L666 -450Q64 -450 61 -448Q55 -446 55 -439Q55 -437 57 -433L590 177Q590 178 557 222T452 366T322 544L56 909L55 924Q55 945 60 948Z'/%3E%3Cpath id='MJMATHI-69' stroke-width='10' d='M184 600Q184 624 203 642T247 661Q265 661 277 649T290 619Q290 596 270 577T226 557Q211 557 198 567T184 600ZM21 287Q21 295 30 318T54 369T98 420T158 442Q197 442 223 419T250 357Q250 340 236 301T196 196T154 83Q149 61 149 51Q149 26 166 26Q175 26 185 29T208 43T235 78T260 137Q263 149 265 151T282 153Q302 153 302 143Q302 135 293 112T268 61T223 11T161 -11Q129 -11 102 10T74 74Q74 91 79 106T122 220Q160 321 166 341T173 380Q173 404 156 404H154Q124 404 99 371T61 287Q60 286 59 284T58 281T56 279T53 278T49 278T41 278H27Q21 284 21 287Z'/%3E%3Cpath id='MJMAIN-3D' stroke-width='10' d='M56 347Q56 360 70 367H707Q722 359 722 347Q722 336 708 328L390 327H72Q56 332 56 347ZM56 153Q56 168 72 173H708Q722 163 722 153Q722 140 707 133H70Q56 140 56 153Z'/%3E%3Cpath id='MJMAIN-30' stroke-width='10' d='M96 585Q152 666 249 666Q297 666 345 640T423 548Q460 465 460 320Q460 165 417 83Q397 41 362 16T301 -15T250 -22Q224 -22 198 -16T137 16T82 83Q39 165 39 320Q39 494 96 585ZM321 597Q291 629 250 629Q208 629 178 597Q153 571 145 525T137 333Q137 175 145 125T181 46Q209 16 250 16Q290 16 318 46Q347 76 354 130T362 333Q362 478 354 524T321 597Z'/%3E%3Cpath id='MJMATHI-6E' stroke-width='10' d='M21 287Q22 293 24 303T36 341T56 388T89 425T135 442Q171 442 195 424T225 390T231 369Q231 367 232 367L243 378Q304 442 382 442Q436 442 469 415T503 336T465 179T427 52Q427 26 444 26Q450 26 453 27Q482 32 505 65T540 145Q542 153 560 153Q580 153 580 145Q580 144 576 130Q568 101 554 73T508 17T439 -10Q392 -10 371 17T350 73Q350 92 386 193T423 345Q423 404 379 404H374Q288 404 229 303L222 291L189 157Q156 26 151 16Q138 -11 108 -11Q95 -11 87 -5T76 7T74 17Q74 30 112 180T152 343Q153 348 153 366Q153 405 129 405Q91 405 66 305Q60 285 60 284Q58 278 41 278H27Q21 284 21 287Z'/%3E%3Cpath id='MJMAIN-28' stroke-width='10' d='M94 250Q94 319 104 381T127 488T164 576T202 643T244 695T277 729T302 750H315H319Q333 750 333 741Q333 738 316 720T275 667T226 581T184 443T167 250T184 58T225 -81T274 -167T316 -220T333 -241Q333 -250 318 -250H315H302L274 -226Q180 -141 137 -14T94 250Z'/%3E%3Cpath id='MJMAIN-2B' stroke-width='10' d='M56 237T56 250T70 270H369V420L370 570Q380 583 389 583Q402 583 409 568V270H707Q722 262 722 250T707 230H409V-68Q401 -82 391 -82H389H387Q375 -82 369 -68V230H70Q56 237 56 250Z'/%3E%3Cpath id='MJMAIN-31' stroke-width='10' d='M213 578L200 573Q186 568 160 563T102 556H83V602H102Q149 604 189 617T245 641T273 663Q275 666 285 666Q294 666 302 660V361L303 61Q310 54 315 52T339 48T401 46H427V0H416Q395 3 257 3Q121 3 100 0H88V46H114Q136 46 152 46T177 47T193 50T201 52T207 57T213 61V578Z'/%3E%3Cpath id='MJMAIN-29' stroke-width='10' d='M60 749L64 750Q69 750 74 750H86L114 726Q208 641 251 514T294 250Q294 182 284 119T261 12T224 -76T186 -143T145 -194T113 -227T90 -246Q87 -249 86 -250H74Q66 -250 63 -250T58 -247T55 -238Q56 -237 66 -225Q221 -64 221 250T66 725Q56 737 55 738Q55 746 60 749Z'/%3E%3Cpath id='MJMAIN-32' stroke-width='10' d='M109 429Q82 429 66 447T50 491Q50 562 103 614T235 666Q326 666 387 610T449 465Q449 422 429 383T381 315T301 241Q265 210 201 149L142 93L218 92Q375 92 385 97Q392 99 409 186V189H449V186Q448 183 436 95T421 3V0H50V19V31Q50 38 56 46T86 81Q115 113 136 137Q145 147 170 174T204 211T233 244T261 278T284 308T305 340T320 369T333 401T340 431T343 464Q343 527 309 573T212 619Q179 619 154 602T119 569T109 550Q109 549 114 549Q132 549 151 535T170 489Q170 464 154 447T109 429Z'/%3E%3Cpath id='MJMAIN-41' stroke-width='10' d='M255 0Q240 3 140 3Q48 3 39 0H32V46H47Q119 49 139 88Q140 91 192 245T295 553T348 708Q351 716 366 716H376Q396 715 400 709Q402 707 508 390L617 67Q624 54 636 51T687 46H717V0H708Q699 3 581 3Q458 3 437 0H427V46H440Q510 46 510 64Q510 66 486 138L462 209H229L209 150Q189 91 189 85Q189 72 209 59T259 46H264V0H255ZM447 255L345 557L244 256Q244 255 345 255H447Z'/%3E%3Cpath id='MJMAIN-64' stroke-width='10' d='M376 495Q376 511 376 535T377 568Q377 613 367 624T316 637H298V660Q298 683 300 683L310 684Q320 685 339 686T376 688Q393 689 413 690T443 693T454 694H457V390Q457 84 458 81Q461 61 472 55T517 46H535V0Q533 0 459 -5T380 -11H373V44L365 37Q307 -11 235 -11Q158 -11 96 50T34 215Q34 315 97 378T244 442Q319 442 376 393V495ZM373 342Q328 405 260 405Q211 405 173 369Q146 341 139 305T131 211Q131 155 138 120T173 59Q203 26 251 26Q322 26 373 103V342Z'/%3E%3Cpath id='MJMAIN-72' stroke-width='10' d='M36 46H50Q89 46 97 60V68Q97 77 97 91T98 122T98 161T98 203Q98 234 98 269T98 328L97 351Q94 370 83 376T38 385H20V408Q20 431 22 431L32 432Q42 433 60 434T96 436Q112 437 131 438T160 441T171 442H174V373Q213 441 271 441H277Q322 441 343 419T364 373Q364 352 351 337T313 322Q288 322 276 338T263 372Q263 381 265 388T270 400T273 405Q271 407 250 401Q234 393 226 386Q179 341 179 207V154Q179 141 179 127T179 101T180 81T180 66V61Q181 59 183 57T188 54T193 51T200 49T207 48T216 47T225 47T235 46T245 46H276V0H267Q249 3 140 3Q37 3 28 0H20V46H36Z'/%3E%3Cpath id='MJMAIN-65' stroke-width='10' d='M28 218Q28 273 48 318T98 391T163 433T229 448Q282 448 320 430T378 380T406 316T415 245Q415 238 408 231H126V216Q126 68 226 36Q246 30 270 30Q312 30 342 62Q359 79 369 104L379 128Q382 131 395 131H398Q415 131 415 121Q415 117 412 108Q393 53 349 21T250 -11Q155 -11 92 58T28 218ZM333 275Q322 403 238 411H236Q228 411 220 410T195 402T166 381T143 340T127 274V267H333V275Z'/%3E%3Cpath id='MJMAIN-73' stroke-width='10' d='M295 316Q295 356 268 385T190 414Q154 414 128 401Q98 382 98 349Q97 344 98 336T114 312T157 287Q175 282 201 278T245 269T277 256Q294 248 310 236T342 195T359 133Q359 71 321 31T198 -10H190Q138 -10 94 26L86 19L77 10Q71 4 65 -1L54 -11H46H42Q39 -11 33 -5V74V132Q33 153 35 157T45 162H54Q66 162 70 158T75 146T82 119T101 77Q136 26 198 26Q295 26 295 104Q295 133 277 151Q257 175 194 187T111 210Q75 227 54 256T33 318Q33 357 50 384T93 424T143 442T187 447H198Q238 447 268 432L283 424L292 431Q302 440 314 448H322H326Q329 448 335 442V310L329 304H301Q295 310 295 316Z'/%3E%3Cpath id='MJMATHI-58' stroke-width='10' d='M42 0H40Q26 0 26 11Q26 15 29 27Q33 41 36 43T55 46Q141 49 190 98Q200 108 306 224T411 342Q302 620 297 625Q288 636 234 637H206Q200 643 200 645T202 664Q206 677 212 683H226Q260 681 347 681Q380 681 408 681T453 682T473 682Q490 682 490 671Q490 670 488 658Q484 643 481 640T465 637Q434 634 411 620L488 426L541 485Q646 598 646 610Q646 628 622 635Q617 635 609 637Q594 637 594 648Q594 650 596 664Q600 677 606 683H618Q619 683 643 683T697 681T738 680Q828 680 837 683H845Q852 676 852 672Q850 647 840 637H824Q790 636 763 628T722 611T698 593L687 584Q687 585 592 480L505 384Q505 383 536 304T601 142T638 56Q648 47 699 46Q734 46 734 37Q734 35 732 23Q728 7 725 4T711 1Q708 1 678 1T589 2Q528 2 496 2T461 1Q444 1 444 10Q444 11 446 25Q448 35 450 39T455 44T464 46T480 47T506 54Q523 62 523 64Q522 64 476 181L429 299Q241 95 236 84Q232 76 232 72Q232 53 261 47Q262 47 267 47T273 46Q276 46 277 46T280 45T283 42T284 35Q284 26 282 19Q279 6 276 4T261 1Q258 1 243 1T201 2T142 2Q64 2 42 0Z'/%3E%3Cpath id='MJMAIN-2C' stroke-width='10' d='M78 35T78 60T94 103T137 121Q165 121 187 96T210 8Q210 -27 201 -60T180 -117T154 -158T130 -185T117 -194Q113 -194 104 -185T95 -172Q95 -168 106 -156T131 -126T157 -76T173 -3V9L172 8Q170 7 167 6T161 3T152 1T140 0Q113 0 96 17Z'/%3E%3Cpath id='MJMATHI-59' stroke-width='10' d='M66 637Q54 637 49 637T39 638T32 641T30 647T33 664T42 682Q44 683 56 683Q104 680 165 680Q288 680 306 683H316Q322 677 322 674T320 656Q316 643 310 637H298Q242 637 242 624Q242 619 292 477T343 333L346 336Q350 340 358 349T379 373T411 410T454 461Q546 568 561 587T577 618Q577 634 545 637Q528 637 528 647Q528 649 530 661Q533 676 535 679T549 683Q551 683 578 682T657 680Q684 680 713 681T746 682Q763 682 763 673Q763 669 760 657T755 643Q753 637 734 637Q662 632 617 587Q608 578 477 424L348 273L322 169Q295 62 295 57Q295 46 363 46Q379 46 384 45T390 35Q390 33 388 23Q384 6 382 4T366 1Q361 1 324 1T232 2Q170 2 138 2T102 1Q84 1 84 9Q84 14 87 24Q88 27 89 30T90 35T91 39T93 42T96 44T101 45T107 45T116 46T129 46Q168 47 180 50T198 63Q201 68 227 171L252 274L129 623Q128 624 127 625T125 627T122 629T118 631T113 633T105 634T96 635T83 636T66 637Z'/%3E%3Cpath id='MJMAIN-22C5' stroke-width='10' d='M78 250Q78 274 95 292T138 310Q162 310 180 294T199 251Q199 226 182 208T139 190T96 207T78 250Z'/%3E%3Cpath id='MJMATHI-53' stroke-width='10' d='M308 24Q367 24 416 76T466 197Q466 260 414 284Q308 311 278 321T236 341Q176 383 176 462Q176 523 208 573T273 648Q302 673 343 688T407 704H418H425Q521 704 564 640Q565 640 577 653T603 682T623 704Q624 704 627 704T632 705Q645 705 645 698T617 577T585 459T569 456Q549 456 549 465Q549 471 550 475Q550 478 551 494T553 520Q553 554 544 579T526 616T501 641Q465 662 419 662Q362 662 313 616T263 510Q263 480 278 458T319 427Q323 425 389 408T456 390Q490 379 522 342T554 242Q554 216 546 186Q541 164 528 137T492 78T426 18T332 -20Q320 -22 298 -22Q199 -22 144 33L134 44L106 13Q83 -14 78 -18T65 -22Q52 -22 52 -14Q52 -11 110 221Q112 227 130 227H143Q149 221 149 216Q149 214 148 207T144 186T142 153Q144 114 160 87T203 47T255 29T308 24Z'/%3E%3Cpath id='MJMATHI-74' stroke-width='10' d='M26 385Q19 392 19 395Q19 399 22 411T27 425Q29 430 36 430T87 431H140L159 511Q162 522 166 540T173 566T179 586T187 603T197 615T211 624T229 626Q247 625 254 615T261 596Q261 589 252 549T232 470L222 433Q222 431 272 431H323Q330 424 330 420Q330 398 317 385H210L174 240Q135 80 135 68Q135 26 162 26Q197 26 230 60T283 144Q285 150 288 151T303 153H307Q322 153 322 145Q322 142 319 133Q314 117 301 95T267 48T216 6T155 -11Q125 -11 98 4T59 56Q57 64 57 83V101L92 241Q127 382 128 383Q128 385 77 385H26Z'/%3E%3Cpath id='MJMATHI-72' stroke-width='10' d='M21 287Q22 290 23 295T28 317T38 348T53 381T73 411T99 433T132 442Q161 442 183 430T214 408T225 388Q227 382 228 382T236 389Q284 441 347 441H350Q398 441 422 400Q430 381 430 363Q430 333 417 315T391 292T366 288Q346 288 334 299T322 328Q322 376 378 392Q356 405 342 405Q286 405 239 331Q229 315 224 298T190 165Q156 25 151 16Q138 -11 108 -11Q95 -11 87 -5T76 7T74 17Q74 30 114 189T154 366Q154 405 128 405Q107 405 92 377T68 316T57 280Q55 278 41 278H27Q21 284 21 287Z'/%3E%3Cpath id='MJMATHI-64' stroke-width='10' d='M366 683Q367 683 438 688T511 694Q523 694 523 686Q523 679 450 384T375 83T374 68Q374 26 402 26Q411 27 422 35Q443 55 463 131Q469 151 473 152Q475 153 483 153H487H491Q506 153 506 145Q506 140 503 129Q490 79 473 48T445 8T417 -8Q409 -10 393 -10Q359 -10 336 5T306 36L300 51Q299 52 296 50Q294 48 292 46Q233 -10 172 -10Q117 -10 75 30T33 157Q33 205 53 255T101 341Q148 398 195 420T280 442Q336 442 364 400Q369 394 369 396Q370 400 396 505T424 616Q424 629 417 632T378 637H357Q351 643 351 645T353 664Q358 683 366 683ZM352 326Q329 405 277 405Q242 405 210 374T160 293Q131 214 119 129Q119 126 119 118T118 106Q118 61 136 44T179 26Q233 26 290 98L298 109L352 326Z'/%3E%3Cpath id='MJMATHI-65' stroke-width='10' d='M39 168Q39 225 58 272T107 350T174 402T244 433T307 442H310Q355 442 388 420T421 355Q421 265 310 237Q261 224 176 223Q139 223 138 221Q138 219 132 186T125 128Q125 81 146 54T209 26T302 45T394 111Q403 121 406 121Q410 121 419 112T429 98T420 82T390 55T344 24T281 -1T205 -11Q126 -11 83 42T39 168ZM373 353Q367 405 305 405Q272 405 244 391T199 357T170 316T154 280T149 261Q149 260 169 260Q282 260 327 284T373 353Z'/%3E%3Cpath id='MJMAIN-53' stroke-width='10' d='M55 507Q55 590 112 647T243 704H257Q342 704 405 641L426 672Q431 679 436 687T446 700L449 704Q450 704 453 704T459 705H463Q466 705 472 699V462L466 456H448Q437 456 435 459T430 479Q413 605 329 646Q292 662 254 662Q201 662 168 626T135 542Q135 508 152 480T200 435Q210 431 286 412T370 389Q427 367 463 314T500 191Q500 110 448 45T301 -21Q245 -21 201 -4T140 27L122 41Q118 36 107 21T87 -7T78 -21Q76 -22 68 -22H64Q61 -22 55 -16V101Q55 220 56 222Q58 227 76 227H89Q95 221 95 214Q95 182 105 151T139 90T205 42T305 24Q352 24 386 62T420 155Q420 198 398 233T340 281Q284 295 266 300Q261 301 239 306T206 314T174 325T141 343T112 367T85 402Q55 451 55 507Z'/%3E%3Cpath id='MJMAIN-74' stroke-width='10' d='M27 422Q80 426 109 478T141 600V615H181V431H316V385H181V241Q182 116 182 100T189 68Q203 29 238 29Q282 29 292 100Q293 108 293 146V181H333V146V134Q333 57 291 17Q264 -10 221 -10Q187 -10 162 2T124 33T105 68T98 100Q97 107 97 248V385H18V422H27Z'/%3E%3Cpath id='MJMAIN-69' stroke-width='10' d='M69 609Q69 637 87 653T131 669Q154 667 171 652T188 609Q188 579 171 564T129 549Q104 549 87 564T69 609ZM247 0Q232 3 143 3Q132 3 106 3T56 1L34 0H26V46H42Q70 46 91 49Q100 53 102 60T104 102V205V293Q104 345 102 359T88 378Q74 385 41 385H30V408Q30 431 32 431L42 432Q52 433 70 434T106 436Q123 437 142 438T171 441T182 442H185V62Q190 52 197 50T232 46H255V0H247Z'/%3E%3Cpath id='MJMAIN-42' stroke-width='10' d='M131 622Q124 629 120 631T104 634T61 637H28V683H229H267H346Q423 683 459 678T531 651Q574 627 599 590T624 512Q624 461 583 419T476 360L466 357Q539 348 595 302T651 187Q651 119 600 67T469 3Q456 1 242 0H28V46H61Q103 47 112 49T131 61V622ZM511 513Q511 560 485 594T416 636Q415 636 403 636T371 636T333 637Q266 637 251 636T232 628Q229 624 229 499V374H312L396 375L406 377Q410 378 417 380T442 393T474 417T499 456T511 513ZM537 188Q537 239 509 282T430 336L329 337H229V200V116Q229 57 234 52Q240 47 334 47H383Q425 47 443 53Q486 67 511 104T537 188Z'/%3E%3Cpath id='MJMAIN-79' stroke-width='10' d='M69 -66Q91 -66 104 -80T118 -116Q118 -134 109 -145T91 -160Q84 -163 97 -166Q104 -168 111 -168Q131 -168 148 -159T175 -138T197 -106T213 -75T225 -43L242 0L170 183Q150 233 125 297Q101 358 96 368T80 381Q79 382 78 382Q66 385 34 385H19V431H26L46 430Q65 430 88 429T122 428Q129 428 142 428T171 429T200 430T224 430L233 431H241V385H232Q183 385 185 366L286 112Q286 113 332 227L376 341V350Q376 365 366 373T348 383T334 385H331V431H337H344Q351 431 361 431T382 430T405 429T422 429Q477 429 503 431H508V385H497Q441 380 422 345Q420 343 378 235T289 9T227 -131Q180 -204 113 -204Q69 -204 44 -177T19 -116Q19 -89 35 -78T69 -66Z'/%3E%3Cpath id='MJMAIN-50' stroke-width='10' d='M130 622Q123 629 119 631T103 634T60 637H27V683H214Q237 683 276 683T331 684Q419 684 471 671T567 616Q624 563 624 489Q624 421 573 372T451 307Q429 302 328 301H234V181Q234 62 237 58Q245 47 304 46H337V0H326Q305 3 182 3Q47 3 38 0H27V46H60Q102 47 111 49T130 61V622ZM507 488Q507 514 506 528T500 564T483 597T450 620T397 635Q385 637 307 637H286Q237 637 234 628Q231 624 231 483V342H302H339Q390 342 423 349T481 382Q507 411 507 488Z'/%3E%3Cpath id='MJMAIN-78' stroke-width='10' d='M201 0Q189 3 102 3Q26 3 17 0H11V46H25Q48 47 67 52T96 61T121 78T139 96T160 122T180 150L226 210L168 288Q159 301 149 315T133 336T122 351T113 363T107 370T100 376T94 379T88 381T80 383Q74 383 44 385H16V431H23Q59 429 126 429Q219 429 229 431H237V385Q201 381 201 369Q201 367 211 353T239 315T268 274L272 270L297 304Q329 345 329 358Q329 364 327 369T322 376T317 380T310 384L307 385H302V431H309Q324 428 408 428Q487 428 493 431H499V385H492Q443 385 411 368Q394 360 377 341T312 257L296 236L358 151Q424 61 429 57T446 50Q464 46 499 46H516V0H510H502Q494 1 482 1T457 2T432 2T414 3Q403 3 377 3T327 1L304 0H295V46H298Q309 46 320 51T331 63Q331 65 291 120L250 175Q249 174 219 133T185 88Q181 83 181 74Q181 63 188 55T206 46Q208 46 208 23V0H201Z'/%3E%3Cpath id='MJMAIN-6C' stroke-width='10' d='M42 46H56Q95 46 103 60V68Q103 77 103 91T103 124T104 167T104 217T104 272T104 329Q104 366 104 407T104 482T104 542T103 586T103 603Q100 622 89 628T44 637H26V660Q26 683 28 683L38 684Q48 685 67 686T104 688Q121 689 141 690T171 693T182 694H185V379Q185 62 186 60Q190 52 198 49Q219 46 247 46H263V0H255L232 1Q209 2 183 2T145 3T107 3T57 1L34 0H26V46H42Z'/%3E%3C/defs%3E%3C/svg%3E"/>

Alas, images are rarely linear in memory. To improve cache
efficiency, modern graphics hardware interleaves the X- and Y-coordinates.
Instead of one row after the next, pixels in memory follow a [spiral-like
curve](https://fgiesen.wordpress.com/2011/01/17/texture-tiling-and-swizzling/).

We need to amend our previous equation to interleave the coordinates. We could
use many instructions to mask one bit at a time, shifting to construct the
interleaved result, but that's inefficient. We can do better.

There is a well-known ["bit twiddling" algorithm to interleave
bits](https://graphics.stanford.edu/~seander/bithacks.html#InterleaveBMN).
Rather than shuffle one bit at a time, the algorithm shuffles groups of bits,
parallelizing the problem. Implementing this algorithm in shader code improves
performance.

In practice, only the lower 7-bits (or less) of each coordinate are
interleaved. That lets us use 32-bit instructions to "vectorize" the
interleave, by putting the X- and Y-coordinates in the low and high 16-bits of
a 32-bit register. Those 32-bit instructions let us interleave X and Y at the
same time, halving the instruction count. Plus, we can exploit the GPU's
combined shift-and-add instruction. Putting the tricks together, we interleave
in 10 instructions of M1 GPU assembly:

```asm
# Inputs x, y in r0l, r0h.
# Output in r1.

add r2, #0, r0, lsl 4
or  r1, r0, r2
and r1, r1, #0xf0f0f0f
add r2, #0, r1, lsl 2
or  r1, r1, r2
and r1, r1, #0x33333333
add r2, #0, r1, lsl 1
or  r1, r1, r2
and r1, r1, #0x55555555
add r1, r1l, r1h, lsl 1
```

We could stop here, but what if there's a *dedicated* instruction to interleave
bits? PowerVR has a "shuffle" instruction
[`shfl`](https://docs.imgtec.com/reference-manuals/powervr-instruction-set-reference/topics/bitwise-instructions/SHFL.html),
and the M1 GPU borrows from PowerVR. Perhaps that instruction was borrowed too.
Unfortunately, even if it was, the proprietary compiler won't use it when
compiling our test shaders. That makes it difficult to reverse-engineer the
instruction -- if it exists -- by observing compiled shaders.

It's time to dust off a powerful reverse-engineering technique from
magic kindergarten: guess and check.

[Dougall Johnson]( https://mastodon.social/@dougall) provided the guess.
When considering the instructions we already know about, he took special notice
of the "reverse bits" instruction. Since reversing bits is a type of bit
shuffle, the interleave instruction should be encoded similarly. The bit
reverse instruction has a two-bit field specifying the operation, with value
`01`. Related instructions to _count the number of set bits_ and _find the
first set bit_ have values `10` and `11` respectively. That encompasses all
known "complex bit manipulation" instructions.

There is one value of the two-bit enumeration that is unobserved and unknown:
`00`. If this interleave instruction exists, it's probably encoded like the bit
reverse but with operation code `00` instead of `01`.

There's a difficulty: the three known instructions have one single input
source, but our instruction interleaves two sources. Where does the second
source go? We can make a guess based on symmetry. Presumably to simplify the
hardware decoder, M1 GPU instructions usually encode their sources
in consistent locations across instructions. The other three instructions have
a gap where we would expect the second source to be, in a two-source
arithmetic instruction. Probably the second source is there.

Armed with a guess, it's our turn to check. Rather than handwrite GPU assembly,
we can hack our compiler to replace some two-source integer operation (like
multiply) with our guessed encoding of "interleave". Then we write a compute
shader using this operation (by "multiplying" numbers) and run it with the
newfangled compute support in our driver.

All that's left is writing a
[shader](/img/blog/2024/02/blog/interleave.shader_test) that checks that
the mystery instruction returns the interleaved result for each possible input.
Since the instruction takes two 16-bit sources, there are about 4 billion
($2^32$) inputs. With our driver, the M1 GPU manages to check them all in under
a second, and the verdict is in: this is our interleave instruction.

As for our clever vectorized assembly to interleave coordinates? We can replace
it with one instruction. It's anticlimactic, but it's fast and it passes
the conformance tests.

And that's what matters.

---

_Thank you to [Khronos](https://www.khronos.org/) and [Software in the Public Interest](https://www.spi-inc.org/) for supporting open
drivers._
