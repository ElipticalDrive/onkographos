# Rendering Pipelines and Passes

Also called: Render Forward Pass

## Overview
Rendering Pipelines and Passes refers to the structural organization and execution sequence of GPU rendering stages used to transform volumetric splats into a final image. It solves the problem of efficiently managing the complex data flow between geometry processing, rasterization, and compositing stages required for real-time volume visualization [S1, S2, unofficial]. It differs from [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md) by relying on hardware rasterization rather than ray casting, and from [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md) by focusing on the static structural definition of passes rather than temporal execution slicing.

## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [S1, S3, S4, unofficial].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1, S3, S4, unofficial]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial][S6, unofficial][S10, abstract only, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6, unofficial][S10, abstract only, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, S2, S3, S4, unofficial]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2, unofficial][S6, unofficial][S10, abstract only, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
    where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [S10, abstract only, unofficial].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, unofficial][S7]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, abstract only, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2, S3, unofficial].

## Subtopics

### [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md)
A modular pipeline interface that decouples the acceleration logic of ray marching and volumetric compositing from the specific radiance field representation. It allows efficient, differentiable volumetric rendering to be integrated into any user-defined neural radiance field by implementing two query functions. Unlike specialized implementations such as Instant-NGP which rely on custom CUDA kernels, this method provides a Python API built on PyTorch, prioritizing ease of extension for dynamic and unbounded scenes over raw kernel-level optimization.

### [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md)
A hardware design methodology for volume rendering systems that achieves real-time performance by overlapping the execution of distinct pipeline stages through time-slicing. It solves the bottleneck of sequential processing in traditional volume rendering pipelines by allowing multiple tasks to be processed simultaneously on shared or replicated functional blocks. Unlike general-purpose DSP-based solutions that require multiple chips for similar volumes, this architecture integrates the rendering pipeline into a unified multi-core structure, enabling continuous hardware-software communication and interactive frame rates.

### [Slice-by-Slice (Sheet Buffer) Splatting](2110-slice-by-slice-sheet-buffer-splatting.md)
A volume rendering optimization that organizes splats into cache-sheets aligned parallel to the volume face most closely parallel to the image plane, then composites these sheets slice-by-slice. It sets itself apart from its siblings, such as [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md) and [Screen Quad Rendering](2435-screen-quad-rendering.md), by leveraging hardware-accelerated rasterization for compositing rather than ray-casting or full-screen quads, while avoiding the "sparkling" artifacts of per-splat compositing at the cost of introducing "popping" artifacts when the view angle crosses 45° relative to the data axes.

### [Screen Quad Rendering](2435-screen-quad-rendering.md)
The final compositing pass in a two-pass volume rendering pipeline, responsible for displaying the pre-computed volumetric data (stored in off-screen textures) onto the default framebuffer. It solves the problem of transferring high-precision or auxiliary data (such as position or depth) from intermediate render targets to the visible screen while applying final transformations like upscaling or debug overlays. It differs from sibling methods like [Fullscreen Triangle Rendering](4230-fullscreen-triangle-rendering.md) by its specific reliance on a two-stage architecture where the first pass handles volume splatting into Framebuffer Objects (FBOs), and this second pass handles the final presentation.

### [Rendering Pipeline Architecture](3059-rendering-pipeline-architecture.md)
Defines the structural organization of GPU rendering stages used to transform volumetric splats into a final image. It solves the problem of efficiently managing the complex data flow between geometry processing, rasterization, and compositing stages required for real-time volume visualization. It differs from [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md) by relying on hardware rasterization rather than ray casting, and from [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md) by focusing on the static structural definition of passes rather than temporal execution slicing.

### [Custom Pass Rendering](3629-custom-pass-rendering.md)
A multi-pass, object-order rendering architecture that decomposes volumetric visualization into a sequence of specialized GPU compute and fragment shader stages. Unlike single-pass ray marchers, it separates geometric reconstruction (depth/position/normal estimation) from shading and post-processing effects (SSAO, shadows, material lookup). It is distinct from [DirectXSplat Renderer API](8504-directxsplat-renderer-api.md) by relying on explicit frame buffer object (FBO) attachments for intermediate data rather than a unified API abstraction, and from [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md) by executing all passes synchronously per frame rather than asynchronously across frames.

### [Multi-View Splatting Pipeline with Fill-Rate Filtering](4689-multi-view-splatting-pipeline-with-fill-rate-filtering.md)
This method generates multi-view 2D images from a 3D point cloud by splatting points onto image planes and filtering out sparse views based on fill rate to ensure structured, high-quality outputs. It differs from standard splatting by integrating a post-rendering quality control step that discards views with insufficient pixel coverage, making it suitable for dataset generation where consistent view density is required.

### [DirectXSplat Renderer API](8504-directxsplat-renderer-api.md)
A C++/Direct3D 12 library for rendering 3D Gaussian Splats, providing a rasterization pipeline that handles scene upload, preparation, rendering, and frame submission with explicit resource management. It differs from general-purpose splatting libraries by enforcing a host-owned device/queue/command list/fence architecture, allowing integration into existing D3D12 applications while managing its own internal COM references and resource transitions. It is optimized for high performance, claiming up to 2.4x faster rendering than `gsplat` in matched-quality benchmarks on NVIDIA hardware.

### [Ray-Centric and Hardware-Accelerated Rendering](8936-ray-centric-and-hardware-accelerated-rendering.md)
Computes the color of each pixel by tracing a ray from the camera through the volume and accumulating density and color along the ray path, typically executed within a GPU fragment shader. It differs from splatting by avoiding the sorting and blending overheads required for transparent primitives, making it suitable for dense volumes, though it is computationally heavier per pixel than texture-based splatting [G]. It preserves internal volume information, unlike isosurface extraction methods such as [Marching Cubes](53-marching-cubes.md).

### [Multi-Pass and Sequential Stage Architectures](10539-multi-pass-and-sequential-stage-architectures.md)
Decomposes volume rendering into distinct, ordered computational phases to manage complexity, memory, or differentiability. Unlike single-pass splatting which projects primitives directly to the screen, these architectures separate data preparation, accumulation, and final compositing into independent passes. This separation allows for complex per-pixel storage (e.g., fragment lists), deferred shading, or differentiable gradient flow that single-pass methods cannot easily support.

### Image-order Splatting
Uses an image-order approach to improve accuracy when using a perspective projection [S2, unofficial].

## Background
The concept of rendering pipelines in volume rendering evolved from early object-order splatting methods. Westover's seminal work on splatting established the foundation for rasterizing volumetric primitives [S2, unofficial]. The specific implementation referred to as "msplat_render" is associated with the `msplat` project, a native Metal implementation for 3D Gaussian Splatting training [S1, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
