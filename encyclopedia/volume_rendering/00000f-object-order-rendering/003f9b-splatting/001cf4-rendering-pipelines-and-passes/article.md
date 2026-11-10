# Rendering Pipelines and Passes

Also called: Render Forward Pass

## Overview
Rendering Pipelines and Passes is a family of object-order rendering methods that decompose volumetric data into geometric primitives (splats) which are transformed, rasterized, and composited to form an image [S1, S3, S4, unofficial]. It solves the performance bottleneck of volume rendering by leveraging hardware-accelerated rasterization and blending, enabling real-time interactivity for dense point sets or voxel data [S1, S2, S3, unofficial]. It differs from ray-casting methods by processing primitives in screen space rather than sampling rays per pixel, and from pure mesh-based rendering by handling implicit or unstructured data via splatting kernels [S10, abstract only, unofficial].

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
Plug-and-Play Ray Marching and Rendering is a modular pipeline interface that decouples the efficient ray marching and differentiable volumetric rendering logic from the specific radiance field representation. It allows users to plug any user-defined radiance field (e.g., MLPs, hash grids) into a standardized acceleration engine by implementing two query functions. Unlike Instant-NGP, which relies on specialized, monolithic CUDA implementations, this method provides a Python API built on PyTorch, facilitating easier extension to dynamic scenes, unbounded environments, and custom radiance field architectures.

### [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md)
The Time-Sliced Pipelined Architecture is a hardware design methodology for volume rendering systems that achieves real-time performance by time-slicing functional blocks to allow simultaneous processing of multiple tasks. It solves the bottleneck of sequential pipeline execution by overlapping stages, enabling continuous hardware-software communication and interactive user interfaces. Unlike general-purpose DSP-based solutions that require multiple chips for similar volumes, this architecture integrates reusable sub-cores into a multi-core structure, often within a single ASIC or FPGA, to handle the shear-warp factorization pipeline efficiently.

### [Slice-by-Slice (Sheet Buffer) Splatting](2110-slice-by-slice-sheet-buffer-splatting.md)
Slice-by-Slice (Sheet Buffer) Splatting is a volume rendering optimization that organizes splats into cache-sheets aligned parallel to the volume face most closely parallel to the image plane, then composites these sheets slice-by-slice. It sets itself apart from its siblings, such as [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md) and [Screen Quad Rendering](2435-screen-quad-rendering.md), by leveraging hardware-accelerated rasterization for bulk accumulation within 2D planes rather than per-pixel ray casting or full-screen quads, while avoiding the "sparkling" artifacts of [Multi-View Splatting Pipeline with Fill-Rate Filtering](4689-multi-view-splatting-pipeline-with-fill-rate-filtering.md) at the cost of potential "popping" artifacts when view angles change.

### [Screen Quad Rendering](2435-screen-quad-rendering.md)
Screen Quad Rendering is the final compositing pass in a two-pass volume rendering pipeline, responsible for displaying the pre-computed volumetric data (stored in off-screen textures) onto the default framebuffer. It solves the problem of transferring high-precision or auxiliary data (such as position or depth) from intermediate render targets to the visible screen while applying final transformations like upscaling or debug overlays. It differs from sibling methods like [Fullscreen Triangle Rendering](4230-fullscreen-triangle-rendering.md) by its specific reliance on a two-stage architecture where the first pass handles volume splatting into Framebuffer Objects (FBOs), and this second pass handles the final presentation.

### [Rendering Pipeline Architecture](3059-rendering-pipeline-architecture.md)
The Rendering Pipeline Architecture is a family of object-order rendering methods that decompose volumetric data into geometric primitives (splats) which are transformed, rasterized, and composited to form an image. It solves the performance bottleneck of volume rendering by leveraging hardware-accelerated rasterization and blending, enabling real-time interactivity for dense point sets or voxel data. It differs from ray-casting methods by processing primitives in screen space rather than sampling rays per pixel, and from pure mesh-based rendering by handling implicit or unstructured data via splatting kernels.

### [Custom Pass Rendering](3629-custom-pass-rendering.md)
Custom Pass Rendering is a multi-pass, object-order rendering architecture that decomposes volumetric visualization into a sequence of specialized GPU compute and fragment shader stages. Unlike single-pass ray marchers, it separates geometric reconstruction (depth/position/normal estimation) from shading and post-processing effects (SSAO, shadows, material lookup). It is distinct from [DirectXSplat Renderer API](8504-directxsplat-renderer-api.md) by relying on explicit frame buffer object (FBO) attachments for intermediate data rather than a unified API abstraction, and from [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md) by executing all passes synchronously per frame rather than asynchronously across frames.

### [Multi-View Splatting Pipeline with Fill-Rate Filtering](4689-multi-view-splatting-pipeline-with-fill-rate-filtering.md)
This method generates multi-view 2D images from a 3D point cloud by splatting points onto image planes and filtering out sparse views based on fill rate to ensure structured, high-quality outputs. It differs from standard splatting by integrating a post-rendering quality control step that discards views with insufficient pixel coverage, making it suitable for dataset generation where consistent view density is required.

### [DirectXSplat Renderer API](8504-directxsplat-renderer-api.md)
DirectXSplat is a C++/Direct3D 12 library for rendering 3D Gaussian Splats, providing a rasterization pipeline that handles scene upload, preparation, rendering, and frame submission with explicit resource management. It is designed as a fast, GPU-accelerated alternative to other Gaussian Splatting renderers, achieving up to 2.4x higher frame rates in matched-quality resident-scene benchmarks compared to [gsplat](981-plug-and-play-ray-marching-and-rendering.md). Unlike [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md) which may abstract away hardware specifics, DirectXSplat exposes a host-owned device, queue, command list, and fence architecture, giving the application full control over the D3D12 submission loop while managing the complexity of Gaussian splat rasterization internally.

### [Ray-Centric and Hardware-Accelerated Rendering](8936-ray-centric-and-hardware-accelerated-rendering.md)
Ray-centric rendering computes the color of each pixel by tracing a ray from the camera through the volume, sampling the density field, and compositing the result along the ray path. It solves the problem of accurate volumetric visualization by avoiding the sorting and blending artifacts inherent in splatting methods, while leveraging GPU hardware acceleration for real-time performance. It differs from its siblings, such as [Splatting](1411-rendering-pipelines-and-passes.md) which projects primitives to the screen, by processing data along the view direction (ray-order) rather than the screen direction (object-order).

### [Multi-Pass and Sequential Stage Architectures](10539-multi-pass-and-sequential-stage-architectures.md)
Multi-Pass and Sequential Stage Architectures decompose volume rendering into discrete, ordered processing steps executed across multiple rendering passes or shader invocations. This approach solves the complexity of integrating diverse operations—such as geometry assembly, shading, lighting, and compositing—by decoupling them into specialized stages. It sets itself apart from single-pass or ray-marching-only methods by allowing intermediate data (e.g., fragment lists, G-buffers) to be stored and processed between stages, enabling complex effects like deferred shading and differentiable optimization.

### Image-order Splatting
Use an image-order approach to improve accuracy when using a perspective projection [S2, unofficial].

## Background
The concept of splatting was introduced by Westover [S2, unofficial]. The term "splat" refers to the projection of a 3D primitive onto the 2D image plane, creating a footprint that is then rasterized. Early implementations focused on accelerating volume rendering by leveraging the speed of hardware rasterization compared to ray casting [S2, unofficial]. The family has evolved to include various optimizations such as sheet-based compositing, hardware-accelerated time-slicing, and modern differentiable rendering pipelines for neural fields [S1, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
