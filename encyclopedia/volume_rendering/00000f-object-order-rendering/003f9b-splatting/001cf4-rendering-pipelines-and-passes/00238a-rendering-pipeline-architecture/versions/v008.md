# Rendering Pipeline Architecture


## Overview
Rendering Pipeline Architecture defines the structural organization of GPU rendering stages used to transform volumetric splats into a final image. It solves the problem of efficiently managing the complex data flow between geometry processing, rasterization, and compositing stages required for real-time volume visualization. It differs from [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md) by relying on hardware rasterization rather than ray casting, and from [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md) by focusing on the static structural definition of passes rather than temporal execution slicing.

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

### [Splat Attributes Pass](1066-splat-attributes-pass.md)
A fragment-shader-based pass that rasterizes splats to accumulate weighted attributes (color, normal, opacity) into a G-Buffer. It solves the problem of correctly blending overlapping transparent splats by discarding occluded fragments and weighting by radial and depth proximity. It differs from [Single-Pass Splat Rendering Pipeline](1254-single-pass-splat-rendering-pipeline.md) by writing to intermediate textures rather than the final buffer, and from [Three-Pass GPU Point Rendering Pipeline](2837-three-pass-gpu-point-rendering-pipeline.md) by using fragment-shader depth calculations instead of pre-sorted geometry passes.

### [Single-Pass Splat Rendering Pipeline](1254-single-pass-splat-rendering-pipeline.md)
A technique combining splat and mesh rendering to enable high-quality shader effects in a single pass. It solves multi-pass blending artifacts by reorganizing data into elevation map patches. Unlike sibling methods that blend in screen space, it interpolates attributes in object space, resulting in a single pass and improved attribute quality.

### [Three-Pass GPU Point Rendering Pipeline](2837-three-pass-gpu-point-rendering-pipeline.md)
A hardware-accelerated method separating depth testing, color accumulation, and normalization into distinct passes. It solves the problem of correctly compositing transparent splats with perspective-correct filtering without explicit sorting. It differs from [Single-Pass Splat Rendering Pipeline](1254-single-pass-splat-rendering-pipeline.md) by decoupling weight accumulation from color blending for exact normalization.

## Background
The concept of rendering pipelines for splatting evolved from early volume visualization techniques that required explicit sorting or ray casting. The shift toward hardware-accelerated rasterization allowed for real-time performance by leveraging the GPU's fixed-function pipeline for depth testing and blending. The specific architectures described here (G-Buffer accumulation, elevation map patches, and multi-pass normalization) represent optimizations developed to address specific limitations of early splatting, such as normalization errors, attribute blurring, and occlusion handling.

## Sources
[S1] Splat Attributes Pass, Single-Pass Splat Rendering Pipeline, and Three-Pass GPU Point Rendering Pipeline notes.
[S2] Shared principles notes.
[S3] Shared principles notes.
[S4] Shared principles notes.
[S6] Shared principles notes.
[S7] Shared principles notes.
[S10] Shared principles notes.
