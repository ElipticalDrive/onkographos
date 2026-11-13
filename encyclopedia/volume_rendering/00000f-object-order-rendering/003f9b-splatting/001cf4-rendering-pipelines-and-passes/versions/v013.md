# Rendering Pipelines and Passes

Also called: Render Forward Pass

## Overview
Rendering Pipelines and Passes refers to the structural organization and execution flow of object-order volume rendering, specifically within the splatting family, where volumetric data is decomposed into primitives and rasterized to form an image [S1, S3, S4, unofficial]. It solves the problem of efficiently rendering large-scale volumetric datasets in real-time by leveraging GPU rasterization pipelines for transformation, projection, and compositing [S1, S2, S3, unofficial]. Unlike ray-marching methods that sample along rays, this family uses screen-space accumulation (compositing along sheets) or specialized multi-stage architectures to handle transparency and depth occlusion [S10, abstract only, unofficial].

## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [G].

1.  **Primitive Representation:** The volume is decomposed into primitives [G]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial][S6, unofficial][S10, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6, unofficial][S10, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane [G]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2, unofficial][S6, unofficial][S10, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
4.  **Compositing:** Contributions are accumulated to form the final image [G]. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, unofficial][S7, unofficial]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2, S3, unofficial].

## Subtopics

### [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md)
A modular pipeline interface that decouples acceleration logic from the radiance field representation, allowing efficient, differentiable volumetric rendering via two query functions. It prioritizes ease of extension for dynamic scenes over raw kernel-level optimization.

### [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md)
A hardware design methodology achieving real-time performance by overlapping distinct pipeline stages through time-slicing on shared or replicated functional blocks. It integrates the rendering pipeline into a unified multi-core structure, enabling continuous hardware-software communication.

### [Slice-by-Slice (Sheet Buffer) Splatting](2110-slice-by-slice-sheet-buffer-splatting.md)
An optimization organizing splats into cache-sheets aligned parallel to the volume face most closely parallel to the image plane, compositing these sheets slice-by-slice. It leverages hardware-accelerated rasterization for compositing, avoiding "sparkling" artifacts of per-splat compositing but introducing "popping" artifacts when the view angle crosses 45° relative to data axes.

### [Screen Quad Rendering](2435-screen-quad-rendering.md)
The final compositing stage in a two-pass splatting pipeline, where pre-rendered volumetric data stored in off-screen Framebuffer Objects (FBOs) is composited into the default framebuffer. It decouples the volume splatting geometry pass from final screen composition, allowing for post-processing effects or debug overlays.

### [Rendering Pipeline Architecture](3059-rendering-pipeline-architecture.md)
The structural organization of hardware-accelerated splatting techniques, decomposing volumetric data into primitives rasterized to form an image. It differs from ray-marching by using screen-space accumulation and from single-pass mesh rendering by explicitly handling transparency through multi-stage accumulation buffers.

### [Custom Pass Rendering](3629-custom-pass-rendering.md)
A multi-pass, object-order architecture decoupling volumetric traversal and shading from the standard graphics pipeline via specialized compute or fragment shaders. It separates expensive volume traversal into a dedicated pass outputting hit positions/normals to screen-space buffers, consumed by subsequent passes for lighting and compositing.

### [Multi-View Splatting Pipeline with Fill-Rate Filtering](4689-multi-view-splatting-pipeline-with-fill-rate-filtering.md)
Generates multi-view 2D images from a 3D point cloud by rasterizing points as splats, then filters views based on pixel coverage density. It explicitly discards views with insufficient fill rate, ensuring structured output quality for downstream tasks.

### [DirectXSplat Renderer API](8504-directxsplat-renderer-api.md)
A C++/Direct3D 12 library for rendering 3D Gaussian Splats, providing a rasterization pipeline with explicit resource management. It enforces a host-owned device/queue/command list/fence architecture, allowing integration into existing D3D12 applications while managing internal COM references.

### [Ray-Centric and Hardware-Accelerated Rendering](8936-ray-centric-and-hardware-accelerated-rendering.md)
Computes pixel color by tracing a ray from the camera through the volume and accumulating density/color, typically in a GPU fragment shader. It avoids sorting and blending overheads of transparent primitives, making it suitable for dense volumes, though computationally heavier per pixel than texture-based splatting [G].

### [Multi-Pass and Sequential Stage Architectures](10539-multi-pass-and-sequential-stage-architectures.md)
Decomposes volume rendering into distinct, ordered computational phases to manage complexity, memory bandwidth, or differentiability. It decouples geometric traversal, data preparation, and optical compositing, allowing specialized optimization of each stage at the cost of increased memory traffic.

### Image-order Splatting
An approach that improves accuracy when using a perspective projection by processing splats in image order rather than object order [S2, unofficial].

## Background
The concept of splatting as a rendering technique was introduced by Westover in 1989, who mapped view-dependent footprints to a generic footprint table to improve rendering speed [S2, unofficial]. The term "Render Forward Pass" and the specific implementation `msplat_render` are associated with the `msplat` project, a 3D Gaussian Splatting training implementation in ~90s on M4 Max using native Metal [S1, unofficial]. The family has evolved to include hardware-accelerated pipelines, time-sliced architectures, and multi-pass custom shader passes to handle complex volumetric data and lighting interactions in real-time [S1, S2, S3, S4, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
