# Two-pass and Multi-pass Techniques


## Overview
Two-pass and multi-pass techniques split the direct volume ray casting workload into multiple rendering passes to isolate distinct computational stages [G]. This family addresses hardware limitations, such as restricted shader instruction counts or the lack of conditional fragment discard, by decomposing complex operations like light attenuation, boundary detection, and ray integration into sequential steps [S1, book][S2]. It differs from single-pass ray marching by trading additional draw calls for the ability to reuse intermediate state stored in 2D render targets and leverage fixed-function hardware blending modes [G].

## Shared principles
All methods in this family build on the core loop structure defined in [Image-order rendering](16-image-order-rendering.md) and [Direct volume ray casting](19-direct-volume-ray-casting.md) [G]. The core computational engine is split across $N$ passes ($N \ge 2$), where intermediate results are persisted in 2D textures (render targets) and accessed in subsequent passes [G].

The general mechanism involves:
1. **State Persistence**: Intermediate state (e.g., ray entry/exit points, accumulated light, accumulated color) is written to 2D textures after each pass and read in the next [G].
2. **Decoupling**: Distinct physical or logical operations are isolated into separate passes, such as separating light attenuation from color compositing, or boundary intersection from ray marching integration [G].
3. **Hardware Blending**: Many techniques rely on hardware-supported blending modes (e.g., 'over', 'under', additive) to accumulate values across passes, avoiding the need for complex atomic operations or conditional shader logic [G].
4. **Geometry Proxy**: Slicing or proxy geometry (e.g., slice planes, bounding box faces) is often used to drive the rendering passes, allowing the GPU rasterizer to handle ray intersection logic implicitly [G].

## Subtopics

### [Half-angle slicing](1255-half-angle-slicing.md)
A two-pass per-slice technique for volumetric shadows that orients slice planes perpendicular to the half-angle vector between the eye and light directions. It decouples light transport from data volume resolution, allowing a single slice to be rendered from both viewpoints. Choose this when high-resolution shadows are needed without pre-computation or 3D shadow map storage, and when view/light angles are not perpendicular.

### [Two-pass volume rendering](1292-two-pass-volume-rendering.md)
Decouples light attenuation computation from color compositing by separating the workload into a light pass and an eye pass. It differs from single-pass ray marching by allowing efficient reuse of precomputed light attenuation and leveraging hardware 'under' blending for front-to-back compositing. Choose this for standard direct volume rendering where light transport is complex or needs to be reused across frames.

### [Two-Pass Ray Marching Volume Rendering](5965-two-pass-ray-marching-volume-rendering.md)
A GPU-based technique that decouples the determination of ray entry and exit points from the actual volume integration. The first pass renders volume bounding box faces to establish depth ranges, and the second pass performs raymarching integration using those ranges. Choose this when efficient handling of complex boundary intersections or features like cutting planes is required, isolating geometric setup from data sampling.

### [Multi-pass GPU Ray-Casting with Early Ray Termination](7009-multi-pass-gpu-ray-casting-with-early-ray-termination.md)
Implements direct volume ray casting on DirectX 9-level hardware (Pixel Shader 2.0) by decomposing ray traversal into multiple rendering passes. It solves the limitation of early fragment shaders (no conditional discard) by using an intermediate "stopping criterion" pass to update the depth buffer and skip terminated rays. Choose this when targeting older hardware or when early ray termination is critical for performance and single-pass conditional logic is unavailable.

### Multi-pass Stencil Occlusion for Volumetric Light Scattering
An implementation approach for fixed-function hardware or Shader Model 2.0 limits, using additive frame-buffer blending over multiple passes with a stencil occlusion method [S1, book]. It is chosen when Shader Model 3.0 is unavailable, as the single-pass version requires more texture samples than SM2.0 allows [S1, book].

### Anti-Aliased Binary Volume Rendering
A method to alleviate aliasing in direct rendering of binary volumes by using iterative 3D texture mapping with trilinear interpolation to generate gray level intensity data around surface boundaries [S2]. It iteratively refines the volume by writing texture-mapped slices back to texture memory [S2]. Choose this for binary volumes where smooth surface boundaries are desired, typically requiring two or three iterations [S2].

### Intra-Node Compositing
Compositing that takes place on the rendering engine. Multi-pass rendering is used. If rendering takes place on the VolumePro 1000 board, it leverages the capability of rendering multiple volumes in a single context with on-board image compositing [S3, unofficial].

### Multi-Pass Multi-Buffer Transparency
A technique using a combination of multiple RGBAZ value storage in the frame buffer and multi-pass rendering to yield correct transparency results with minimum performance hit [S4, book]. Storing more than one set of RGBAZ values is costly due to additional memory requirements and is limited by the number of values the hardware can store [S4, book].

## Background
The concept of decoupling volume rendering stages emerged with the rise of programmable GPUs, which allowed complex state management via render targets [G]. Early implementations on Shader Model 2.0 hardware necessitated multi-pass approaches due to limited instruction counts and texture sample limits, leading to techniques like multi-pass ray casting with early ray termination [S1, book]. Anti-aliased binary rendering was developed to handle the aliasing issues inherent in direct 3D texture mapping of binary volumes [S2]. These techniques are utilized in applications such as surgical simulation and biomedical data exploration where performance constraints or specific visual effects require decomposition of the rendering pipeline [S2].

## Sources
[S1] Volumetric Light Scattering as a Post-Process, https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-13-volumetric-light-scattering-post-process, fetched 2026-10-27 (book)
[S2] Fast CSG voxelization by frame buffer pixel mapping, https://doi.org/10.1145/353888.353896, fetched 2026-11-04
[S3] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S4] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
