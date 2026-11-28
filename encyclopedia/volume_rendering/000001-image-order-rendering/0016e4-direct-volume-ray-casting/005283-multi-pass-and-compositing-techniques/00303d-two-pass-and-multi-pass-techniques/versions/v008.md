# Two-pass and Multi-pass Techniques

Also called: Multi-pass rendering

## Overview
Two-pass and multi-pass techniques decompose the volume rendering pipeline into sequential rendering passes, using 2D textures (render targets) to persist intermediate state such as ray boundaries, light attenuation, or accumulation buffers between passes [G]. This approach solves the limitations of early GPU hardware (e.g., lack of dynamic flow control or limited texture sampling) and optimizes performance by decoupling complex logical operations—such as ray intersection, light transport, and compositing—into specialized stages [G]. It differs from single-pass methods by introducing a data dependency between passes, and from sibling techniques like [Image-Order Volumetric Compositing](1390-image-order-volumetric-compositing.md) by explicitly managing state persistence across distinct rasterization events rather than relying on a single integrated shader loop [G].

## Shared principles
All methods in this family build on the core loop structure defined in [Image-order rendering](16-image-order-rendering.md) and [Direct volume ray casting](19-direct-volume-ray-casting.md) [G]. The core computational engine is split across $N$ passes ($N \ge 2$), where intermediate results are persisted in 2D textures (render targets) and accessed in subsequent passes [G].

The general mechanism involves:
1. **State Persistence**: Intermediate state (e.g., ray entry/exit points, accumulated light, accumulated color) is written to 2D textures after each pass and read in the next [G].
2. **Decoupling**: Distinct physical or logical operations are isolated into separate passes, such as separating light attenuation from color compositing, or boundary intersection from ray marching integration [G].
3. **Hardware Blending**: Many techniques rely on hardware-supported blending modes (e.g., 'over', 'under', additive) to accumulate values across passes, avoiding the need for complex atomic operations or conditional shader logic [G].
4. **Geometry Proxy**: Slicing or proxy geometry (e.g., slice planes, bounding box faces) is often used to drive the rendering passes, allowing the GPU rasterizer to handle ray intersection logic implicitly [G].

## Subtopics

### [Two-pass volume rendering](1292-two-pass-volume-rendering.md)
Two-pass volume rendering is a direct volume ray casting technique that separates light attenuation computation from color compositing into two distinct rasterization passes. It differs from single-pass methods by decoupling these operations into a two-stage data dependency between a dedicated light buffer and the frame buffer.

### [Two-Pass Ray Marching Volume Rendering](5965-two-pass-ray-marching-volume-rendering.md)
Two-Pass Ray Marching Volume Rendering is a GPU-based direct volume rendering technique that decouples the determination of ray entry and exit points from the actual volume integration (ray marching). It solves the problem of efficiently computing ray-volume intersections without requiring complex ray-triangle intersection logic in the main rendering pass by rasterizing the volume's bounding box faces into offscreen textures. Unlike single-pass raycasting, which must compute intersections within the fragment shader, this method uses hardware rasterization for boundary detection, reducing fragment shader complexity at the cost of an additional render pass and texture memory overhead.

### [Locally Adaptive Volume Rendering](13045-locally-adaptive-volume-rendering.md)
Locally Adaptive Volume Rendering (LAVR) is a two-pass technique that enhances feature visibility and reduces noise in volume rendering by dynamically adjusting fragment opacity based on the local density of contributing voxels. Unlike standard direct volume rendering which uses a fixed per-voxel opacity, LAVR decouples the classification of relevant data from the final compositing, allowing the renderer to adaptively suppress noise in sparse regions while preserving sharp borders in dense regions. It sets itself apart from siblings like [Two-pass volume rendering](1292-two-pass-volume-rendering.md) by specifically targeting the statistical distribution of contributions per pixel rather than just splitting light transport or ray marching steps [G].

### [Multi-pass multi-view PBR](23108-multi-pass-multi-view-pbr.md)
Multi-pass multi-view PBR is a rendering approach that generates $N$ distinct perspective views of a 3D dataset by executing $N$ separate rendering passes, storing each view in a render target texture, and subsequently compositing them into a final spatial image. It is primarily used for auto-stereoscopic displays (e.g., lenticular sheets or wavelength-selective filters) requiring multiple viewpoints. It differs from single-pass multi-view techniques (such as [Half-angle slicing](1255-half-angle-slicing.md) or [Two-pass volume rendering](1292-two-pass-volume-rendering.md)) by isolating each view into its own rasterization pass, which simplifies the shader logic but incurs a significant performance penalty proportional to $N$.

### Multi-pass Stencil Occlusion for Volumetric Light Scattering
#### definition
- [S2, book] An implementation approach for fixed-function hardware or Shader Model 2.0 limits, using additive frame-buffer blending over multiple passes with a stencil occlusion method.
#### parameter
- [S2, book] Shader Model 3.0 is required for the single-pass version because the number of texture samples needed exceeds the limits of Shader Model 2.0.

### Anti-Aliased Binary Volume Rendering
#### definition
- [S3] A method to alleviate aliasing in direct rendering of binary volumes by using iterative 3D texture mapping with trilinear interpolation to generate gray level intensity data around surface boundaries.
#### step
- [S3] Render the binary volume using 3D texture mapping, which uses trilinear interpolation for sample points around surface boundaries, generating intensity values between 0 and 1.
- [S3] Iteratively repeat the process: Write each texture-mapped slice (without blending) from the frame buffer back to the texture memory as a slice of a refined volume.
#### parameter
- [S3] Iterations: Two or three iterations are normally sufficient for reasonably smooth surfaces, with higher rendering cost for more iterations.

### Intra-Node Compositing
#### definition
- [S4, unofficial] Compositing that takes place on the rendering engine. Multi-pass rendering is used. If rendering takes place on the VolumePro 1000 board, it leverages the capability of rendering multiple volumes in a single context with on-board image compositing [S4, unofficial].

### Multi-Pass Multi-Buffer Transparency
#### definition
- [S5, book] A technique using a combination of multiple RGBAZ value storage in the frame buffer and multi-pass rendering to yield correct transparency results with minimum performance hit.
#### pitfall
- [S5, book] Storing more than one set of RGBAZ values is costly due to additional memory requirements and is limited by the number of values the hardware can store.

### Multi-Pass Reflection/Refraction
#### step
- [S6] Render the heterogeneous scene (containing both surface and volume objects) twice.
- [S6] First pass: render from the imaginary eye position.
- [S6] Second pass: render from the actual eye position.
- [S6] Combine the results of both passes to accomplish special effects such as reflection and refraction.

### Multisampled DVR Pass
#### definition
- [S7] A pass that renders actors participating in the RENDER_DVR pass, currently only used by the Volume raycaster.
#### step
- [S7] Set the current render step to RENDER_DVR using MSceneViewGLWidget::renderStep().
#### pitfall
- [S7] Overlapping volumes are composited incorrectly; a warning is logged if more than one actor is rendered in this pass.

### Depth Prepass (Carmack's Reverse)
#### definition
- [S8, unofficial] A two-pass rendering technique to optimize complex shading by first filling the depth buffer with opaque geometry without writing to the color buffer, then rendering the color pass with depth testing enabled but depth writing disabled, preventing expensive shader computations for fragments that will be occluded.
#### step
- [S8, unofficial] Pass 1: Render the scene geometry without writing to the color buffer, only updating the depth buffer.
- [S8, unofficial] Pass 2: Disable depth buffer writing. Render the scene again, writing only to the color buffer while checking the depth buffer.
#### pitfall
- [S8, unofficial] This technique avoids expensive shader computations for fragments that are later overwritten by closer fragments, but requires two full passes over the geometry.
#### history
- [S8, unofficial] Used in Quake 3 by Carmack.

## Background
The concept of splitting rendering tasks across multiple passes originated from the need to handle transparency and complex lighting on early graphics hardware that lacked the computational power or shader flexibility for single-pass solutions [G]. Techniques like multi-pass multi-buffer transparency were developed to address the accumulation of round-off errors in 8-bit frame buffers [S5, book]. Multi-pass approaches offer a higher flexibility, however, they also have a significant management overhead compared to single-pass rendering (i.e., context switching, final compositing) and usually result in lower performance [S1]. Furthermore, optimization techniques like early ray termination are not trivial in multi-pass rendering and create an additional overhead [S1]. Therefore, most state-of-the-art ray-guided volume renderers use single-pass rendering [S1]. A limitation of single-pass approaches is the requirement for the entire working set to fit into the cache; one way to circumvent this requirement is to use single-pass rendering as long as the working set fits into the cache, and to switch to multi-pass rendering when the working set gets too large [S1].

## Sources
[S1] State-of-the-Art in GPU-Based Large-Scale Volume Visualization, https://vcg.seas.harvard.edu/publications/20150513-stateof-the-art-in-gpu-based-large-scale-volume-visualization/paper, fetched 2026-10-24
[S2] Volumetric Light Scattering as a Post-Process, https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-13-volumetric-light-scattering-post-process, fetched 2026-10-27 (book)
[S3] Fast CSG voxelization by frame buffer pixel mapping, https://doi.org/10.1145/353888.353896, fetched 2026-11-04
[S4] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S5] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S6] Appendix A. Volume Rendering Examples, https://techpubs.jurassic.nl/library/manuals/3000/007-3720-002/sgi_html/apa.html, fetched 2026-10-30
[S7] Rendering, https://met3d.readthedocs.io/en/latest/05_developer_manual/04_rendering/index.html, fetched 2026-10-27
[S8] OpenGL voxel engine slow, https://stackoverflow.com/questions/4563693/opengl-voxel-engine-slow, fetched 2026-10-26 (unofficial)
