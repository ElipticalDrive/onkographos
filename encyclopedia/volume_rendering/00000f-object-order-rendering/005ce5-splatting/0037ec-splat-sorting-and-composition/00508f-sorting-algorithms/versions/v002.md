# Sorting Algorithms

Also called: Splat Sorting, Depth Sorting, Global Splat Sorting

## Overview
Sorting Algorithms in volume rendering splatting reorder volumetric primitives (typically 3D Gaussian splats) by depth to ensure correct front-to-back alpha blending [S1, unofficial][S2, unofficial]. This solves the order-dependent transparency problem inherent in splatting, where incorrect draw order leads to visual artifacts [S1, unofficial][S2, unofficial]. Unlike siblings that may use partial ordering or hardware-specific tricks, this family focuses on generating a strict depth-sorted index buffer for the rasterizer [S1, unofficial].

## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [S1][S3, unofficial][S4].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1][S3, unofficial][S4]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1][S6, unofficial][S10, abstract only, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, unofficial][S4]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6, unofficial][S10, abstract only, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1][S2, S3, unofficial][S4]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2, unofficial][S6, unofficial][S10, abstract only, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
    where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [S10, abstract only, unofficial].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1][S2, S3, unofficial][S7]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, abstract only, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1][S2, S3, unofficial].

## Subtopics

### [CPU-Based Splat Sorting](2097-cpu-based-splat-sorting.md)
Offloads depth sorting to the CPU to avoid GPU compute stalls, trading sorting latency for rendering stability. Suitable when GPU compute is unavailable or underutilized.

### [Cascaded 2-way Merge for Global Splat Sorting](7689-cascaded-2-way-merge-for-global-splat-sorting.md)
Merges pre-sorted lists from multiple renderers into a global order using a GPU-based merge tree. Chosen for multi-scene rendering to reduce overdraw and ensure global depth consistency.

### [Bitonic Sort](14217-bitonic-sort.md)
A data-independent GPU network sort providing exact depth ordering per frame. Preferred over partial-order methods for strict transparency requirements.

## Background
Originating from the need to resolve order-dependent transparency in splat-based volume rendering, these algorithms evolved alongside 3D Gaussian Splatting [S1, unofficial][S2, unofficial]. Early implementations relied on CPU sorting due to GPU limitations, while modern approaches leverage GPU subgroups and merge networks for real-time global sorting [S1, unofficial].

## Sources
