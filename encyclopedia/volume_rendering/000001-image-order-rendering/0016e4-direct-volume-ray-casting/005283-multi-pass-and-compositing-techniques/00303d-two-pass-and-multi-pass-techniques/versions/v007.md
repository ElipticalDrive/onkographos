# Two-pass and Multi-pass Techniques

Also called: Multi-pass Volume Rendering, Split-Pass Rendering

## Overview
Two-pass and multi-pass techniques decompose the volume rendering pipeline into sequential rendering passes, using 2D textures (render targets) to persist intermediate state such as ray boundaries, light attenuation, or accumulation buffers between passes [G]. This approach solves the limitations of early GPU hardware (e.g., lack of dynamic flow control or limited texture sampling) and optimizes performance by decoupling complex logical operations—such as ray intersection, light transport, and compositing—into specialized stages [G]. It differs from single-pass methods by introducing a data dependency between passes, and from sibling techniques like [Image-Order Volumetric Compositing](1390-image-order-volumetric-compositing.md) by explicitly managing state persistence across distinct rasterization events rather than relying on a single integrated shader loop.

## Shared principles
All methods in this family build on the core loop structure defined in [Image-order rendering](16-image-order-rendering.md) and [Direct volume ray casting](19-direct-volume-ray-casting.md) [G]. The core computational engine is split across $N$ passes ($N \ge 2$), where intermediate results are persisted in 2D textures (render targets) and accessed in subsequent passes [G].

The general mechanism involves:
1. **State Persistence**: Intermediate state (e.g., ray entry/exit points, accumulated light, accumulated color) is written to 2D textures after each pass and read in the next [G].
2. **Decoupling**: Distinct physical or logical operations are isolated into separate passes, such as separating light attenuation from color compositing, or boundary intersection from ray marching integration [G].
3. **Hardware Blending**: Many techniques rely on hardware-supported blending modes (e.g., 'over', 'under', additive) to accumulate values across passes, avoiding the need for complex atomic operations or conditional shader logic [G].
4. **Geometry Proxy**: Slicing or proxy geometry (e.g., slice planes, bounding box faces) is often used to drive the rendering passes, allowing the GPU rasterizer to handle ray intersection logic implicitly [G].

## Subtopics

### [Half-angle slicing](1255-half-angle-slicing.md)
Half-angle slicing is a two-pass per-slice rendering technique that synchronizes the slicing axis with the illumination computation to integrate volumetric lighting, including hard shadows and indirect scattering, within interactive applications. It solves the problem of computing accurate volumetric shadows without the memory overhead of pre-computed 3D shadow maps by orienting slice planes perpendicular to the half-angle vector between the eye and light directions. This orientation allows a single slice to be rendered from both the observer's and the light source's viewpoints, decoupling light transport from data volume resolution. Unlike standard view-aligned slicing, it minimizes the number of slices required to resolve lighting details such as specular highlights and avoids attenuation leakage artifacts common in 2D shadow buffer approaches.

### [Two-pass volume rendering](1292-two-pass-volume-rendering.md)
Two-pass volume rendering is a direct volume ray casting technique that decouples light attenuation computation from color compositing into two distinct rasterization passes. It solves the performance bottleneck of calculating complex illumination for every pixel in real-time by pre-computing light accumulation in a dedicated light buffer during the first pass, then using this data for a fast, real-time rendering pass. It differs from standard direct volume ray casting by moving the expensive light integration step off the critical rendering path, and from single-pass methods by requiring a two-stage data dependency between the light buffer and the frame buffer.

### [Two-Pass Ray Marching Volume Rendering](5965-two-pass-ray-marching-volume-rendering.md)
Two-Pass Ray Marching Volume Rendering is a GPU-based direct volume rendering technique that decouples the determination of ray entry and exit points from the actual volume integration (ray marching). It solves the problem of efficiently computing ray-volume intersections without requiring complex ray-triangle intersection logic in the main rendering pass by rasterizing the volume's bounding box faces into offscreen textures. Unlike single-pass raycasting, which must compute intersections within the fragment shader, this method uses hardware rasterization for boundary detection, reducing fragment shader complexity at the cost of an additional render pass and texture memory overhead.

### [Multi-pass GPU Ray-Casting with Early Ray Termination](7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md)
Multi-pass GPU Ray-Casting is a stream-based rendering technique that implements direct volume ray casting by splitting the ray traversal loop across multiple rendering passes, using 2D texture render targets to persist intermediate ray state. It solves the limitation of early GPU hardware (specifically Pixel Shader 2.0 era) which lacked dynamic flow control, preventing single-pass ray marchers from terminating rays early. By decoupling ray integration from termination checks, it achieves significant performance gains over unaccelerated ray casting and slice-based volume rendering (SBVR) through early ray termination and empty-space skipping.

### [Locally Adaptive Volume Rendering](13045-locally-adaptive-volume-rendering.md)
Locally Adaptive Volume Rendering (LAVR) is a two-pass technique that enhances feature visibility and reduces noise in volume rendering by dynamically adjusting fragment opacity based on the local density of contributing voxels. Unlike standard direct volume rendering which uses a fixed per-voxel opacity, LAVR decouples the classification of relevant data from the final compositing, allowing the renderer to adaptively suppress noise in sparse regions while preserving sharp borders in dense regions. It sets itself apart from siblings like [Two-pass volume rendering](1292-two-pass-volume-rendering.md) by specifically targeting the statistical distribution of contributions per pixel rather than just splitting light transport or ray marching steps [G].

### Multi-pass Stencil Occlusion for Volumetric Light Scattering
#### definition
- [S1, book] An implementation approach for fixed-function hardware or Shader Model 2.0 limits, using additive frame-buffer blending over multiple passes with a stencil occlusion method.
#### parameter
- [S1, book] Shader Model 3.0 is required for the single-pass version because the number of texture samples needed exceeds the limits of Shader Model 2.0.

### Anti-Aliased Binary Volume Rendering
#### definition
- [S2] A method to alleviate aliasing in direct rendering of binary volumes by using iterative 3D texture mapping with trilinear interpolation to generate gray level intensity data around surface boundaries.
#### step
- [S2] Render the binary volume using 3D texture mapping, which uses trilinear interpolation for sample points around surface boundaries, generating intensity values between 0 and 1.
- [S2] Iteratively repeat the process: Write each texture-mapped slice (without blending) from the frame buffer back to the texture memory as a slice of a refined volume.
#### parameter
- [S2] Iterations: Two or three iterations are normally sufficient for reasonably smooth surfaces, with higher rendering cost for more iterations.

### Intra-Node Compositing
#### definition
- [S3, unofficial] Compositing that takes place on the rendering engine. Multi-pass rendering is used. If rendering takes place on the VolumePro 1000 board, it leverages the capability of rendering multiple volumes in a single context with on-board image compositing.

### Multi-Pass Multi-Buffer Transparency
#### definition
- [S4, book] A technique using a combination of multiple RGBAZ value storage in the frame buffer and multi-pass rendering to yield correct transparency results with minimum performance hit.
#### pitfall
- [S4, book] Storing more than one set of RGBAZ values is costly due to additional memory requirements and is limited by the number of values the hardware can store.

### Multi-Pass Reflection/Refraction
#### step
- [S5] Render the heterogeneous scene (containing both surface and volume objects) twice.
- [S5] First pass: render from the imaginary eye position.
- [S5] Second pass: render from the actual eye position.
- [S5] Combine the results of both passes to accomplish special effects such as reflection and refraction.

## Background
The concept of splitting rendering tasks across multiple passes originated from the need to handle transparency and complex lighting on early graphics hardware that lacked the computational power or shader flexibility for single-pass solutions. Techniques like multi-pass multi-buffer transparency were developed to address the accumulation of round-off errors in 8-bit frame buffers [S4, book]. The specific application of multi-pass techniques to volume rendering, such as decoupling ray marching or light transport, emerged with the advent of programmable GPUs, allowing developers to bypass hardware limitations like the lack of dynamic flow control in Pixel Shader 2.0 [S3, unofficial]. These methods are widely used in medical visualization, scientific data exploration, and real-time rendering engines where performance optimization and visual fidelity trade-offs are critical.

## Sources
[S1] Volumetric Light Scattering as a Post-Process, https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-13-volumetric-light-scattering-post-process, fetched 2026-10-27 (book)
[S2] Fast CSG voxelization by frame buffer pixel mapping, https://doi.org/10.1145/353888.353896, fetched 2026-11-04
[S3] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S4] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S5] Appendix A. Volume Rendering Examples, https://techpubs.jurassic.nl/library/manuals/3000/007-3720-002/sgi_html/apa.html, fetched 2026-10-30
