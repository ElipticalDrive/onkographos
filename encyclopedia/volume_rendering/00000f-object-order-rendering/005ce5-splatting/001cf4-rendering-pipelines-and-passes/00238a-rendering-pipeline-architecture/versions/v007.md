# Rendering Pipeline Architecture


## Overview
Rendering Pipeline Architecture defines the structural organization of GPU-based splatting systems, determining how volumetric primitives are transformed, rasterized, and composited into a final image. It solves the problem of efficiently rendering large-scale point cloud or volumetric data by leveraging hardware-accelerated rasterization rather than software ray-tracing. It differs from [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md) by using screen-space rasterization instead of ray-marching, and from [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md) by typically executing all stages in a single frame or fixed sequence rather than distributing work across time slices.

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
A fragment-shader-based pass that rasterizes splats to accumulate weighted attributes (color, normal, opacity) into a G-Buffer. It solves occlusion by discarding fragments behind the front surface and weighting by radial/depth proximity. Choose when intermediate attribute buffers are needed for subsequent lighting/compositing.

### [Single-Pass Splat Rendering Pipeline](1254-single-pass-splat-rendering-pipeline.md)
Combines splat and mesh rendering in one pass using object-space attribute interpolation. It avoids multi-pass artifacts and supports high-quality shading (reflection/refraction). Choose when high-fidelity shader effects are required without the overhead of multiple passes.

### [Three-Pass GPU Point Rendering Pipeline](2837-three-pass-gpu-point-rendering-pipeline.md)
Separates depth occlusion, color accumulation, and normalization into three distinct stages. It solves correct blending of overlapping splats on standard hardware without software sorting. Choose when precise weight normalization and occlusion culling are critical.

## Background
Not given in the sources.

## Sources
