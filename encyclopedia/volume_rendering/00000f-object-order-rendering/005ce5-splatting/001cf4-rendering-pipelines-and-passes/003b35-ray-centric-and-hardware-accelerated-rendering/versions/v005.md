# Ray-Centric and Hardware-Accelerated Rendering

Also called: GPU Raycasting, Single-Pass Raycasting, Hardware-Accelerated Ray Tracing

## Overview
Ray-centric rendering computes the color of each pixel by tracing a ray from the camera through the volume, sampling the density field, and compositing the result along the ray path [S2]. Unlike splatting methods which project 3D primitives onto the 2D screen and blend them [Parent](1411-rendering-pipelines-and-passes.md), ray-centric methods evaluate the volume integral directly per pixel, offering higher accuracy for complex lighting and transparency at the cost of higher computational intensity per sample [S2]. It is distinguished from [Batched Ray Rendering](1547-batched-ray-rendering.md) by its execution model: while batched rendering manages memory constraints by chunking ray batches for inference, ray-centric rendering defines the fundamental geometric traversal and shading logic executed on the GPU [S2].

## Shared principles
The family shares the core mechanism of object-order evaluation where each pixel (or ray) is processed independently or in parallel groups to compute the volume rendering equation [S2].

1.  **Ray Generation:** For each pixel $(x,y)$ in the viewport, a ray is defined by an origin $\mathbf{o}$ and direction $\mathbf{d}$ in world space. The origin is typically the camera position, and the direction is computed by unprojecting the pixel coordinates through the inverse View-Projection matrix [S2].
2.  **Ray-Volume Intersection:** The ray is tested against the volume's bounding box (AABB) to determine the entry ($t_0$) and exit ($t_1$) points. This is commonly done using the slab method [S2]:
    $$ \mathbf{t}_{top} = \frac{\text{box.top} - \mathbf{o}}{\mathbf{d}} $$
    $$ \mathbf{t}_{bottom} = \frac{\text{box.bottom} - \mathbf{o}}{\mathbf{d}} $$
    $$ t_0 = \max(0.0, \max(\min(\mathbf{t}_{top}, \mathbf{t}_{bottom}).x, \min(\mathbf{t}_{top}, \mathbf{t}_{bottom}).y)) $$
    $$ t_1 = \min(\max(\mathbf{t}_{top}, \mathbf{t}_{bottom}).x, \max(\mathbf{t}_{top}, \mathbf{t}_{bottom}).y) $$
    [S2].
3.  **Sampling and Transfer Function:** The ray is marched from $t_0$ to $t_1$ with a fixed or adaptive step size. At each step, the density $\rho$ is sampled from a 3D texture or neural field. A transfer function maps $\rho$ to color $C$ and opacity $\alpha$ [S2].
4.  **Front-to-Back Compositing:** Contributions are accumulated using the standard volume rendering equation:
    $$ C_{out} = \sum_{i=1}^{N} T_i \cdot \alpha_i \cdot C_i $$
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $T_i$ is the transmittance at step $i$ [S2].

## Subtopics

### [Batched Ray Rendering](1547-batched-ray-rendering.md)
A memory-management strategy that processes large batches of rays in fixed-size chunks to prevent Out-Of-Memory (OOM) errors on the GPU. It is primarily used during validation and inference phases where full-resolution images must be generated without training-time optimizations. It differs from [Chunked Trilinear Volume Rendering](2557-chunked-trilinear-volume-rendering.md) by chunking the *rays* rather than the *volume data*. See [Batched Ray Rendering](1547-batched-ray-rendering.md) for implementation details.

### [GPU Implementation Pipeline](2385-gpu-implementation-pipeline.md)
A GPU-accelerated technique that traces rays from the camera through each pixel into a 3D volume to compute color and opacity, implemented in a single pass using fragment shaders. It leverages hardware rasterization for ray-box intersection and texture sampling, avoiding the overhead of intermediate scratch buffers or complex sorting required by splatting methods. Unlike [Batched Ray Rendering](1547-batched-ray-rendering.md), this pipeline typically processes pixels or small batches in a direct fragment shader execution model. See [GPU Implementation Pipeline](2385-gpu-implementation-pipeline.md) for implementation details.

### OptiX Volume Rendering Pipeline
A volume rendering method implemented using NVIDIA OptiX. It uses a Ray Generation shader, Miss shaders for radiance and shadow, and Hit/Any-Hit/Intersection shaders for volume traversal and shading [S1, unofficial].
*   **Initialization:** Set `shader_raygen` to `__raygen__default`. Add miss shaders `__miss__radiance` and `__miss__shadow` [S1, unofficial].
*   **Configuration:** Set `group.type = VOLUME_STRUCTURED_REGULAR`. Assign hit group shaders: Closest Hit `__closesthit__volume_radiance` and `__closesthit__volume_shadow`, Any Hit `__anyhit__volume_radiance` and `__anyhit__volume_shadow`, and Intersection `__intersection__volume` [S1, unofficial].

### Hybrid Splatting/Tracing Rendering
A rendering pipeline that rasterizes primary rays for speed and traces secondary rays for lighting effects (reflections/refractions) within the same 3D representation [S2].
*   **Primary Rays:** Compute all primary ray intersections with the scene, render these using rasterization, and discard Gaussian hits that fall behind a ray's closest intersection [S2].
*   **Secondary Rays:** Compute and trace secondary rays using 3DGRT to enable complex visual effects like reflections and refractions [S2].

## Background
The method originates from the fundamental volume rendering equation [G]. Hardware-accelerated raycasting became feasible with the advent of programmable fragment shaders (GLSL/HLSL) in the mid-2000s, allowing per-pixel ray marching on the GPU [S2]. Recent developments integrate these techniques with neural fields and Gaussian splatting, enabling hybrid rendering pipelines that combine rasterization speed with ray-tracing accuracy [S2].

## Sources
[S1] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
[S2] 3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02425, fetched 2026-10-26
