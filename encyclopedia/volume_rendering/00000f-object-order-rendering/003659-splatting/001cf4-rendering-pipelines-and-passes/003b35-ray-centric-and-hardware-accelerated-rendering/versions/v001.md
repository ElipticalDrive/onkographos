# Ray-Centric and Hardware-Accelerated Rendering

Also called: GPU Ray Casting, Hardware-Accelerated Volume Rendering

## Overview
Ray-centric rendering computes the color of each pixel by tracing a ray from the camera through the volume and accumulating contributions from sampled points along the ray path [S2, unofficial]. It differs from object-order splatting by avoiding the sorting and blending overheads required for dense volumes, though it is computationally heavier per pixel than texture-based splatting [G]. It preserves internal volume information, distinguishing it from isosurface extraction methods like [Marching Cubes](53-marching-cubes.md) [S1, unofficial].

## Shared principles
The family operates on the principle of per-pixel ray traversal, where the fragment shader or compute kernel acts as the primary driver for volume integration.

1.  **Ray Generation:** For each pixel (or sample), a ray is defined by an origin $O$ and direction $D$. The origin is typically the camera position, and the direction is computed by inverting the view-projection matrix to map screen coordinates to world space [S2, unofficial].
2.  **Bounding Box Intersection:** The ray is tested against an Axis-Aligned Bounding Box (AABB) enclosing the volume to determine the entry ($t_0$) and exit ($t_1$) distances. This is commonly done using the slab method, handling zero-direction components via reciprocal infinity [S2, unofficial].
3.  **Ray Marching:** The ray is discretized into steps of length $step\_length$. Positions are converted to texture coordinates to sample the 3D volume texture [S2, unofficial].
4.  **Front-to-Back Compositing:** Contributions are accumulated using alpha blending. For each sample, the opacity $\alpha$ and color $C$ are computed via a transfer function. The accumulated color $C_{acc}$ and opacity $A_{acc}$ are updated as:
    $$
    \begin{aligned}
    C_{acc} &= C_{acc} + (1 - A_{acc}) \cdot \alpha \cdot C \\
    A_{acc} &= A_{acc} + (1 - A_{acc}) \cdot \alpha
    \end{aligned}
    $$
    [S2, unofficial]
5.  **Termination:** Marching stops when the ray exits the volume or the accumulated opacity exceeds a threshold (e.g., 1.0), optimizing performance by skipping background samples [S2, unofficial].

## Subtopics

### [Batched Ray Rendering](1547-batched-ray-rendering.md)
Batched Ray Rendering is a memory-management pipeline for volume rendering that splits large ray batches into fixed-size chunks to prevent GPU Out-Of-Memory (OOM) errors. It orchestrates the sequential execution of coarse and fine network passes per chunk, distinguishing itself from full pipeline descriptions by isolating the batching mechanism (`batchify_rays`) and per-chunk rendering logic (`render_rays`). See [Batched Ray Rendering](1547-batched-ray-rendering.md).

### [GPU Implementation Pipeline](2385-gpu-implementation-pipeline.md)
A GPU-accelerated rendering technique that traces rays from the camera through each pixel into a 3D volume to compute color and opacity, implemented in a single pass using fragment shaders. It solves the problem of visualizing volumetric data directly on the GPU without generating intermediate polygonal meshes. See [GPU Implementation Pipeline](2385-gpu-implementation-pipeline.md).

### OptiX Volume Rendering Pipeline
A volume rendering method implemented using NVIDIA OptiX. It uses a Ray Generation shader, Miss shaders for radiance and shadow, and Hit/Any-Hit/Intersection shaders for volume traversal and shading [S1, unofficial].
*   **Initialization:** Set `shader_raygen` to `__raygen__default`. Add miss shaders `__miss__radiance` and `__miss__shadow` [S1, unofficial].
*   **Configuration:** Set `group.type = VOLUME_STRUCTURED_REGULAR`. Assign hit group shaders: Closest Hit `__closesthit__volume_radiance` and `__closesthit__volume_shadow`, Any Hit `__anyhit__volume_radiance` and `__anyhit__volume_shadow`, and Intersection `__intersection__volume` [S1, unofficial].

## Background
The approach leverages the parallel processing power of GPUs to perform volume integration directly in the fragment shader, moving away from CPU-based ray casting. Early implementations utilized fixed-function pipelines or early fragment shaders, evolving to use 3D textures and hardware-accelerated ray-box intersection logic. The method is widely used in medical imaging and scientific visualization where real-time interaction with dense volumetric data is required.

## Sources
[S1] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
