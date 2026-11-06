# Hierarchical and Multi-Resolution Splatting

Also called: Multi-Resolution Splatting, Progressive Splatting

## Overview
Hierarchical and Multi-Resolution Splatting is a family of object-order volume rendering techniques that accelerate rendering by decomposing the volume into a multi-resolution data structure, allowing the renderer to adapt the sampling density to the desired image quality or error threshold. It sets itself apart from standard splatting by explicitly constructing a hierarchy (such as a pyramid or tree) to skip low-detail regions, and from [EWA Splatting](1069-ewa-splatting.md) by focusing on spatial coarsening rather than just kernel shape distortion.

## Shared principles
The family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [S1, S3, S4, unofficial].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1, S3, S4, unofficial]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial][S6][S10, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6][S10, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, S2, S3, S4, unofficial]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2, unofficial][S6][S10, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
    where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [S10, unofficial].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, S3, unofficial][S7, book]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2, S3, unofficial].

## Subtopics

### [Hierarchical Splatting](1168-hierarchical-splatting.md)
Hierarchical splatting is a progressive refinement algorithm for volume rendering that uses a pyramidal volume representation to achieve real-time interactive rendering. It solves the problem of rendering large datasets efficiently by allowing the user to start with a low-resolution approximation and progressively refine the image as needed. It sets itself apart from standard splatting by using a multi-resolution pyramid structure to adapt the rendering resolution to the desired image quality, and from [Multi-Resolution Points](4784-multi-resolution-points.md) by explicitly constructing a hierarchical pyramid of averaged volume cells rather than just selecting points based on density or distance. See [Hierarchical Splatting](1168-hierarchical-splatting.md).

### [Multi-Resolution Points](4784-multi-resolution-points.md)
Multi-Resolution Points is a hierarchical splatting technique that renders a single large point in place of many small points to reduce rasterization cost and storage requirements. It differs from standard splatting by dynamically controlling point size based on local volume content and error metrics, rather than using a fixed resolution. This builds on [Hierarchical and Multi-Resolution Splatting](5618-hierarchical-and-multi-resolution-splatting.md). See [Multi-Resolution Points](4784-multi-resolution-points.md).

### [Splat List (SVL) Generation with Multi-Level Downsampling](5767-splat-list-svl-generation-with-multi-level-downsampling.md)
A rendering technique that generates a sparse voxel list (SVL) of splat geometry across multiple levels of detail (LOD) by iteratively downsampling a voxel occupancy mask and extracting splat primitives for each level.

### RBF Splatting with Octree and Fragment Programs
- [S1, unofficial] A splatting implementation using Rational Basis Functions (RBFs) located by an octree, evaluated via fragment programs on cache-sheets.
- [S1, unofficial] Add an octree to locate the Rational Basis Functions (RBFs) intersecting the cache-sheets.
- [S1, unofficial] For each sheet plane pixel, use a fragment program to evaluate the exponential Gaussian function of the RBFs.
- [S1, unofficial] Performance: 7 to 70 fps for Blunt-fin rendering at 400^2 resolution, caused by large variation of RBFs. Hardware: 2.8 GHz Pentium-IV, nVidia GeForce FX 5900.

## Background
The concept of using multi-resolution representations for volume rendering dates back to the mid-1990s. Cabral et al. [Cabral94] were among the first to demonstrate that texture capabilities could be leveraged for volume rendering, laying groundwork for later hierarchical approaches. The specific application of hierarchical pyramids to splatting was developed to address performance bottlenecks in large dataset visualization. Jang et al. [Jang04] later extended these concepts by integrating octrees with Rational Basis Functions (RBFs) and utilizing GPU fragment programs for evaluation, achieving frame rates ranging from 7 to 70 fps for specific datasets like the Blunt-fin model on hardware such as the nVidia GeForce FX 5900 [S1, unofficial]. Neophytou et al. [Neophytou06] further advanced the field by using floating-point rasterization facilities to avoid expensive fragment programs, achieving near-interactive rates on more powerful hardware [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
