# Understanding Tiled Mobile GPUs through VR Volume Rendering

## NOTE: This is a WIP, the structure and data are due to change in the coming weeks; and suggestions/comments/corrections are always appreciated  :)
## The final publication with the results and conclusions is [here](https://repositori.upf.edu/handle/10230/58025), as the final project for my computer engineering degree.

## The Goal and Why

My goal is to better understand the strengths and limitations of the tiled GPU architecture, under the lens of stereoscopic volume rendering. In order to achieve this I intend to do empirical testing via the use of performance profilers and multiple tests, on the Meta Quest 2 platform, powered by Qualcomm Adreno Graphics.

Qualcomm is one of the biggest players in the mobile GPU hardware space, supplying them for one of the most widespread devices nowadays, the Meta Quest 2 VR headset from Meta. However, its ecosystem is very closed, and very little documentation is available to the public.

This hardware is usually quite different from their desktop counterparts, with a distinct (tiled) infrastructure.The hardware is not just a less powerful version of their desktop counterparts; it brings is own set of constraints to the table. 

Via the use of empirical testing & debugging tools, I plan to better my understanding of this architecture, through the lens of stereoscopic volume rendering.

## The methodology

The main idea is to assess the cost of different rendering techniques, and try to remedy the issues of their implementation, and how they scale, the more percentage of the viewport is filled with the volume rendering in question.

The techniques present some old approaches, revisited in this paradigm; some usual suspects; and more modern ones:

* [Raymarching](https://github.com/JsMarq96/Understanding-Tileg-GPUs-VR-Volume-Rendering/blob/main/raymarching/raymarching.md): a classical approach, representing ray-based methods.
* [Mipmap Accelerated Raymarching](https://github.com/JsMarq96/Understanding-Tileg-GPUs-VR-Volume-Rendering/blob/main/mipmap-accel-raymarching/mar.md): modern acceleration on top of classical raymarching.
* [Volumetric billboards](https://github.com/JsMarq96/Understanding-Tiled-GPUs-VR-Volume-Rendering/blob/main/volumetric-billboard/billboards.md): an unused approach, revisited for mobile.
* [Surface nets](https://github.com/JsMarq96/Understanding-Tiled-GPUs-VR-Volume-Rendering/blob/main/surface-nets/surface_nets.md): representing mesh generation methods.
* Empty Space Skipping: a combination of mesh generation & raymarching, for less iterations.
* Octree based rendering: an attempt to use pure virtual geometry for rendering.

The volume that is going to be presented is the ubiquitous bonsai 3D texture, and I am going to center in rendering an Isosurface, since when trying to achieve a volumetric or spectral representation there is not a lot of maneuverability with the cost.

If time allows, It would also be interesting how effective can be the different more traditional VR techniques on top of this list; like Foveated Rendering.

Using the RenderDoc Meta Fork, it is possible to obtain quite a lot of low-level information, in a per-drawcall matter, such as:

* Frame times
* Clocks
* Texture cache misses: L1, L2, total rates
* Total number of bytes read from memory
* ALU usage per vertex/fragment
* EFU usage per vertex/fragment
* Fragments shaded
* Usage per viewport tile

This can help paint the cost of each different technique; in different viewport configurations.

In order to reduce run-to-run variance, the perspective for each test will be fixed, and the results will be averaged between multiple samples.

The tests will be done in different fixed perspectives; and a final free-roam test, for an overall experience. The fixed perspectives differ on the scale of the rendered volume in the viewport. This helps establish the overall cost in a per-pixel basis.

(TODO add the viewport examples)

## The platform

Since there is a lot of variability between manufacturers of mobile GPUs, I decided to go with the most ubiquitous one, Qualcomm and their Snapdragon XR2 SOC; in the Meta Quest 2 device.

My reasoning for this is straightforward: it is the most ubiquitous and accessible XR device, and that allowed Qualcomm to stay as the de-facto chip maker for XR mobile-powered devices; and researching volume rendering in this application can bring benefits to more cost effective VR medical platforms.

This poses some problems, like the absolute absence of official documentation of the SOC, best practices, and this sort of techniques.

It also offers some interesting capabilities, in the form of Qualcomm's exclusive extensions, for Variable Rate Shading, Frame Reprojecting, Foveated Rendering, and so.

## The test & development environment

When working with a standalone headset like the Quest 2, remote development can be a challenge. The debugging utilities are limited, with a (comparatively) huge time to deploy, when compared to desktop development.

Taking advantage of the cross-platform nature of OpenGL, the development was done on a desktop system with the following specs:


| Processor | i5-12600KF     |
| ----------- | ---------------- |
| GPU       | RTX 3080       |
| RAM       | 16 GB 4800 MHz |

This also helped to establish a comparative baseline of performance.

The test environment is a Meta Quest 2, running the software version XXX. The performance indicator for both GPU & CPU are set to maximum.

The main environment is in OpenGL ES 3.3, due its comparable features to OpenGL 4.0. Since these tests are not in a complex scene, with different materials and meshes, I determined that the benefits of working in Vulkan in this set of tests could be negligible (at least in most tests). Most of the techniques require one draw-call per eye.

## Tiled GPUs

One of the fundamental problems of mobile GPUs is power consumption and thermal efficiency. One of the biggest culprits of this is the high bandwidth required for communication between the different parts of the SoC (System on Chip). This was pointed out by Qualcomm as an approximate of 16% of the total device power consumption, in the 2015 SIGGRAPH talk about Mobile hardware and Bandwidth:

![Slide from the Qualcomm presentation: https://community.arm.com/cfs-file/__key/communityserver-blogs-components-weblogfiles/00-00-00-20-66/siggraph2015_2D00_mmg_2D00_andy_2D00_slides.pdf](https://github.com/JsMarq96/Understanding-Tiled-GPUs-VR-Volume-Rendering/blob/main/imgs/power-consumption.PNG?raw=true)

In order to fix that, they proposed the use of a Tiled Architecture for the GPU. The GPU will split the framebuffer in bins, and compute them within a tiny high-speed on chip local memory for each bin (called GMEM in Snapdragon Adreno GPUs).

When rendering, the triangles are batched per bin, and evaluated all at once. This differs from the immediate mode rendering, of submitting a triangle & rendering it directly, as seen on desktop GPUs.

This limits the data traffic to the system memory, reducing the need for a high-speed large memory. This results in better power consumption and efficiency, but with downsides. When submitting a render pass, the system needs to copy the resulting framebuffer from each bin, in the GMEM, to system memory. This is called "resolving a frame", and adds a non-negligible cost on submitting a frame.

This means that more normal rendering approaches are prohibitive.
For example, using a (normal) deferred pipeline is out of the question, since for that to work, there is a need to resolve many times for a frame.

(https://github.com/mems/calepin/blob/main/Graphics/Graphics.md & https://www.youtube.com/watch?v=SeySx0TkluE)

## Techniques index
* [State of the Art & terminology](https://github.com/JsMarq96/Understanding-Tiled-GPUs-VR-Volume-Rendering/blob/main/vr-state-of-the-art.md)
* [Raymarching](https://github.com/JsMarq96/Understanding-Tileg-GPUs-VR-Volume-Rendering/blob/main/raymarching/raymarching.md)
* [Mipmap Accelerated Raymarching](https://github.com/JsMarq96/Understanding-Tileg-GPUs-VR-Volume-Rendering/blob/main/mipmap-accel-raymarching/mar.md)
* [Surface nets](https://github.com/JsMarq96/Understanding-Tiled-GPUs-VR-Volume-Rendering/blob/main/surface-nets/surface_nets.md)
* [Volumetric billboards](https://github.com/JsMarq96/Understanding-Tiled-GPUs-VR-Volume-Rendering/blob/main/volumetric-billboard/billboards.md)

## Repositories

* [Desktop, OpenGL 4.0 implementation](https://github.com/JsMarq96/Volume-Rendering-Desktop)
* [Quest, OpenGL ES 3.3 implementation](https://github.com/JsMarq96/Quest-Tiled-Volume-Rendering)

## References

* [GPU Gems 2: Per pixel displacement mapping with Distance functions](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-8-pixel-displacement-mapping-distance-functions)
* [Indirect Illumination with Efficient Monte Carlo Denoising](https://link.springer.com/article/10.1007/s11042-020-09884-5)
* [Qualcomm's Official Snapdragon Mobile Platform OpenCL Optimization Guide](https://developer.qualcomm.com/download/adrenosdk/adreno-opencl-programming-guide.pdf?referrer=node/6114https:/)
* [ARM Mali Best Practices](https://documentation-service.arm.com/static/62f4f9b7c3b04f2bd53e1c65)
* [Qualcomm Adreno best Practices](https://developer.qualcomm.com/sites/default/files/docs/adreno-gpu/snapdragon-game-toolkit/gdg/gpu/best_practices.html)
* [Volumetric Billboards](https://hal.inria.fr/inria-00402067)
* [GigaVoxels: ray-guided streaming for efficient and detailed voxel rendering](https://dl.acm.org/doi/10.1145/1507149.1507152)
* [Constrained Elastic Surface Nets:generating smooth surfaces from binary segmented data](https://www.merl.com/publications/docs/TR99-24.pdf)
* [Smooth Voxel Terrain part 2](https://0fps.net/2012/07/12/smooth-voxel-terrain-part-2/)
* [Learning from Failure (SIGGRAPH 2015), by Alex Evans of Media Molecule](https://advances.realtimerendering.com/s2015/AlexEvans_SIGGRAPH-2015-sml.pdf)
* [Compute for Mobile Devices (SIGGRAPH ASIA 2014), by Maxim Shevtson from Intel](https://kesen.realtimerendering.com/Compute_for_Mobile_Devices5.pdf)
* [Understanding and resolving Graphics Memory Loads](https://developer.qualcomm.com/sites/default/files/docs/adreno-gpu/snapdragon-game-toolkit/gdg/tutorials/android/gmem_loads.html)
* [Mobile hardware and bandwidth (SIGGRAPH 2015), by Andrew Gruber from Qualcomm](https://community.arm.com/cfs-file/__key/communityserver-blogs-components-weblogfiles/00-00-00-20-66/siggraph2015_2D00_mmg_2D00_andy_2D00_slides.pdf)
