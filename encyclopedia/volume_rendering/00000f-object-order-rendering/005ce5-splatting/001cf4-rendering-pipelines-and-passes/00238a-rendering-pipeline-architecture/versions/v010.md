# Rendering Pipeline Architecture


## Overview
Rendering Pipeline Architecture refers to the structural organization of hardware-accelerated splatting techniques, where volumetric or point-cloud data is decomposed into primitives (splats) and rasterized to form an image. It solves the problem of efficiently rendering large-scale volumetric datasets in real-time by leveraging GPU rasterization pipelines for transformation, projection, and compositing. It differs from ray-marching methods by using screen-space accumulation (compositing along sheets) rather than per-ray sampling, and from single-pass mesh rendering by explicitly handling transparency and depth occlusion through multi-stage or specialized accumulation buffers.

## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized.
1. **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian).
2. **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp). For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening.
3. **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction:
$$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
The intensity written to the pixel is modulated by the voxel density:
$$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
where $\text{weight}(x,y) = F(x-x_D, y-y_D)$.
4. **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time).
5. **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity.

## Subtopics

### [Splat Attributes Pass](1066-splat-attributes-pass.md)
A fragment-processing stage that reconstructs continuous and cardinal attributes (normals, colors) into a G-Buffer using weighted blending and ternary depth tests. Choose this when accurate per-splat attribute reconstruction with transparency is required.

### [Single-Pass Splat Rendering Pipeline](1254-single-pass-splat-rendering-pipeline.md)
Combines splat and mesh rendering to enable high-quality shader effects in a single pass by interpolating attributes in object space via elevation map patches. Choose this when avoiding multi-pass blending artifacts and blurring is critical.

### [Three-Pass GPU Point Rendering Pipeline](2837-three-pass-gpu-point-rendering-pipeline.md)
Separates depth testing, color accumulation, and normalization into distinct passes to ensure exact perspective-correct filtering without explicit sorting. Choose this when exact weight normalization and correct occlusion handling are prioritized over single-pass performance.

## Background
The concept of splatting was introduced by Levoy as a method for volume visualization. The family evolved to address the limitations of early software-based rendering by leveraging GPU hardware acceleration. Subsequent developments focused on improving attribute quality [1066-splat-attributes-pass.md], reducing pass counts [1254-single-pass-splat-rendering-pipeline.md], and ensuring numerical correctness in perspective projection [2837-three-pass-gpu-point-rendering-pipeline.md].

## Sources
