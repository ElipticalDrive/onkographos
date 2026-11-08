# Splat Sorting and Composition

Also called: Splat Sorting, Splat Composition, Order-Dependent Splatting, Order-Independent Splatting

## Overview
Splat Sorting and Composition is the process of ordering volumetric primitives (splats) by depth relative to the camera and accumulating their contributions to form a final image. It solves the problem of correct transparency rendering, where the visual result depends on the order in which overlapping transparent surfaces are blended. It is distinct from ray-casting methods by operating on a discrete set of projected primitives rather than continuous volume samples, and from order-independent transparency (OIT) techniques by relying on explicit depth sorting to ensure physical correctness, albeit at the cost of sorting overhead.

## Shared principles
The Splat Sorting and Composition family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [S1, S3, S4, unofficial].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1, S3, S4, unofficial]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial][S6][S10, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6][S10, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, S2, S3, S4, unofficial]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2, unofficial][S6][S10, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
    where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [S10, unofficial].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, S2, unofficial][S3][S7, book]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, S2, unofficial][S3].

## Subtopics

### [Stop-the-pop](1472-stop-the-pop.md)
Stop-the-pop is a technique to mitigate popping artifacts in volume splatting that are caused by the sorting procedure of Gaussians along the ray. It uses hierarchical or pixel-resorting to remove popping artifacts without sacrificing speed. It is distinct from methods that modify opacity compensation terms or use ray-tracing-style intersections for accurate depth. See [Stop-the-pop](1472-stop-the-pop.md).

### [CPU-Based Splat Sorting](2097-cpu-based-splat-sorting.md)
CPU-Based Splat Sorting is a method for reordering 3D Gaussian splats by depth on the central processing unit to enable correct front-to-back alpha blending for transparency. It solves the problem of order-independent transparency in volume rendering by decoupling the sorting phase from the GPU rendering pipeline, allowing the GPU to focus exclusively on rasterization and blending. It differs from GPU-based sorting methods (like bitonic sort) by avoiding the overhead of complex GPU sorting algorithms and instead relying on CPU-side sorting algorithms to produce a sorted order buffer. See [CPU-Based Splat Sorting](2097-cpu-based-splat-sorting.md).

### [Order Independent Transparency for Point-Based Models](2150-order-independent-transparency-for-point-based-models.md)
This method renders transparent point-based models on GPUs in an order-independent manner by extending depth-peeling techniques to deferred-shaded multi-pass splatting. It iteratively reconstructs a fixed number of transparent layers from closest to farthest, avoiding explicit depth sorting at the cost of potential color loss if the layer limit is exceeded. See [Order Independent Transparency for Point-Based Models](2150-order-independent-transparency-for-point-based-models.md).

### [Cascaded 2-way Merge for Global Splat Sorting](7689-cascaded-2-way-merge-for-global-splat-sorting.md)
A GPU-based algorithm to merge sorted splat lists from multiple active renderers into a single globally sorted order buffer using a cascaded binary merge tree. See [Cascaded 2-way Merge for Global Splat Sorting](7689-cascaded-2-way-merge-for-global-splat-sorting.md).

### Splat-Every-Sample Composition
#### definition
- [S1, unofficial] A composition method where every sample is splatted, compared against slice-by-slice (sheet buffer) splatting.
#### comparison
- [S1, unofficial] Splat-every-sample produces darker images due to incorrect visibility determination. It achieves better performance by avoiding per-slice additions.

### Order Independent EWA Volume Splatting
#### definition
- [S2, unofficial] A novel order independent EWA volume splatting method that unifies the advantages of previous OI techniques and provides an efficient high-speed approximation by exploiting the analytical expression of elliptical Gaussian kernels used for the noise.
#### step
- [S2, unofficial] Compute a fast view-dependent approximation of accumulated densities along viewing directions using a per-pixel projective volume rendering approach.
- [S2, unofficial] Project the spatially bound kernel functions directly (splat them) thanks to their analytic representation.
#### comparison
- [S2, unofficial] Unlike ray marching which requires hundreds of evaluations per pixel and is computationally demanding for procedural noise, this method provides interactive visualization.
- [S2, unofficial] Unlike EWA volume splatting which requires time-consuming back-to-front ordering and introduces rasterization overhead due to overlapping ellipses, this OI method gets rid of ordering to trade realism for speed.

### Object-level rendering
#### definition
- [S3] A method to segment splats by object and prioritize them based on their summed contribution, allowing high-quality representation for important objects and low-quality for others.
#### step
- [S3] Segment splats belonging to specific objects (manually or via 2D segmentation methods).
- [S3] Find the contribution of splats within this segmentation to determine the importance of objects.
- [S3] Prioritize objects based on their contribution to the scene, sending higher-quality representations of important objects.

## Background
Splatting was introduced as a hardware-accelerated volume rendering technique to leverage the 2D rasterization capabilities of GPUs [S1, S2, unofficial][S3]. Early implementations relied on back-to-front sorting to ensure correct alpha blending, which became a bottleneck as the number of primitives increased [S2, unofficial]. Subsequent research focused on optimizing this sorting process, leading to CPU-based offloading [S1, unofficial][S2, unofficial], hierarchical sorting to reduce artifacts [S2, unofficial][S3], and order-independent techniques that trade some visual fidelity for performance [S2, unofficial]. The advent of 3D Gaussian Splatting has renewed interest in efficient sorting and composition strategies for large-scale point-based volume representations [S3].

## Sources
[S1] Hardwareaccelerated adaptive ewa volume splatting, https://core.ac.uk/works/16783031, fetched 2026-10-28 (unofficial)
[S2] Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis, https://hal.science/hal-02413269v1, fetched 2026-10-26 (unofficial)
[S3] PRoGS: Progressive Rendering of Gaussian Splats, https://doi.org/10.1109/wacv61041.2025.00308, fetched 2026-10-26
