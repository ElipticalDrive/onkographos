# Ray-Centric and Hardware-Accelerated Rendering

Also called: GPU Raycasting, Single-Pass Raycasting, Hardware-Accelerated Ray Tracing

## Overview
This family encompasses volume rendering techniques that compute pixel colors by tracing rays from the camera through the volume data, leveraging GPU hardware acceleration for performance. Unlike splatting methods that rasterize primitives, ray-centric methods sample the volume along the view direction, offering higher fidelity for complex lighting and transparency [S2]. It distinguishes itself from sibling methods like [Slice-by-Slice Splatting](2110-slice-by-slice-sheet-buffer-splatting.md) by processing rays individually or in batches rather than projecting sheets, and from [Batched Ray Rendering](1547-batched-ray-rendering.md) by focusing on the general hardware-accelerated execution model rather than just memory management [S2].

## Shared principles
The family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [S1, S3, S4, unofficial].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1, S3, S4, unofficial]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial][S6, unofficial][S10, abstract only, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6, unofficial][S10, abstract only, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, S2, S3, S4, unofficial]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2][S6, unofficial][S10, abstract only, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
    where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [S10, abstract only, unofficial].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, unofficial][S7]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, abstract only, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2, S3, unofficial].

## Subtopics

### [Batched Ray Rendering](1547-batched-ray-rendering.md)
A memory-management pipeline that splits large ray batches into fixed-size chunks to prevent GPU Out-Of-Memory (OOM) errors. It orchestrates sequential coarse and fine network passes per chunk. Choose this when rendering large volumes with neural networks where memory is constrained.

### [GPU Implementation Pipeline](2385-gpu-implementation-pipeline.md)
A technique that traces rays from the camera through each pixel into a 3D volume to compute color and opacity, implemented in a single pass using fragment shaders. It solves real-time visualization by leveraging hardware rasterization for ray-box intersection and texture sampling, avoiding intermediate scratch buffers. Choose this for standard real-time volume visualization without neural network overhead.

### OptiX Volume Rendering Pipeline
- **Definition:** A volume rendering method implemented using NVIDIA OptiX that uses a Ray Generation shader, Miss shaders for radiance and shadow, and Hit/Any-Hit/Intersection shaders for volume traversal and shading [S1, unofficial].
- **Steps:** Initialize shaders by setting `shader_raygen` to `__raygen__default` and adding miss shaders `__miss__radiance` and `__miss__shadow`. Configure the volume object group with `group.type = VOLUME_STRUCTURED_REGULAR`. Assign hit group shaders: Closest Hit `__closesthit__volume_radiance` and `__closesthit__volume_shadow`, Any Hit `__anyhit__volume_radiance` and `__anyhit__volume_shadow`, and Intersection `__intersection__volume` [S1, unofficial].

### Hybrid Splatting/Tracing Rendering
- **Definition:** A rendering pipeline that rasterizes primary rays for speed and traces secondary rays for lighting effects (reflections/refractions) within the same 3D representation [S2].
- **Steps:** Compute all primary ray intersections with the scene. Render these primary rays using rasterization (splatting). Discard Gaussian hits that fall behind a ray's closest intersection. Compute and trace secondary rays using 3DGRT [S2].

## Background
The concept of hardware-accelerated ray casting for volume rendering emerged in the early 2000s as GPU capabilities grew. Early implementations utilized fragment shaders to perform ray-box intersection and marching loops directly on the GPU, enabling real-time frame rates for large datasets [S2]. The development of specialized APIs like NVIDIA OptiX provided a structured pipeline for ray tracing, allowing for more complex shading and shadowing calculations [S1, unofficial]. More recently, hybrid approaches have combined the speed of rasterization for primary rays with the accuracy of ray tracing for secondary effects, leveraging modern Gaussian Splatting techniques [S2].

## Sources
[S1] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
[S2] 3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02425, fetched 2026-10-26
