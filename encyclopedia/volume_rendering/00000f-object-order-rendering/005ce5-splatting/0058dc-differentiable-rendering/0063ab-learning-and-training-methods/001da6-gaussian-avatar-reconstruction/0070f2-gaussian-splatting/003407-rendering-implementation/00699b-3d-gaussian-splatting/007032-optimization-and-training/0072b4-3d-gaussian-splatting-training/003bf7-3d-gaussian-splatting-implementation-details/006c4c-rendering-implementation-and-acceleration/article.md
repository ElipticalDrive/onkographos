# Rendering Implementation and Acceleration

Also called: 3D Gaussian Splatting Rendering Pipeline, 3DGS Rasterization

## Overview
Rendering Implementation and Acceleration defines the core computational pipeline for projecting a set of $N$ 3D Gaussian primitives $\{G_i\}_{i=1}^N$ onto a 2D image plane and accumulating their contributions via alpha blending. It solves the problem of efficiently rendering millions of anisotropic, view-dependent splats in real-time by leveraging GPU parallelism through tiling, sorting, and culling. It sets itself apart from sibling methods like [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md) by focusing on runtime execution speed rather than data size, and from [Occupancy Region Derivation & ϵ-Intersection Criterion](19020-occupancy-region-derivation-intersection-criterion.md) by using standard bounding-box culling rather than shape-aware elliptic intersection tests.

## Shared principles
All methods in this family operate on the set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$ defined in [Rendering Implementation and Acceleration](21069-rendering-implementation-and-acceleration.md). The core rendering operation involves projecting these 3D primitives to 2D screen space and performing alpha blending. Acceleration relies on the following shared mechanisms:

1.  **Tiling:** Dividing the image plane into non-overlapping tiles (commonly $16 \times 16$ pixels) to localize Gaussian processing and enable coalesced memory accesses [S2, unofficial].
2.  **Sorting:** Sorting Gaussians within each tile by view-space depth (front-to-back) to enable correct alpha blending [S2, unofficial].
3.  **Culling:** Discarding Gaussians that do not overlap with the camera frustum or specific tiles to reduce enumeration overhead [S2, unofficial].
4.  **Memory Hierarchy Management:** Exploiting the distinction between global memory (VRAM), shared memory (on-chip), and host memory (CPU RAM) to minimize data movement.

## Subtopics

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
This family reduces the storage footprint and transmission bandwidth of 3D Gaussian Splatting (3DGS) models by compressing geometric attributes (position, scale, rotation), opacity, and view-dependent color (Spherical Harmonics coefficients). It solves the bottleneck of large model sizes (often hundreds of MBs) which hinders deployment on mobile devices or web platforms. It is distinguished from sibling methods like [Hardware and Computational Acceleration](16214-hardware-and-computational-acceleration.md) by focusing on data representation size rather than rendering speed, and from [Occupancy Region Derivation & ϵ-Intersection Criterion](19020-occupancy-region-derivation-intersection-criterion.md) by explicitly quantizing or clustering attributes rather than just culling.

### [Hardware and Computational Acceleration](16214-hardware-and-computational-acceleration.md)
Hardware and Computational Acceleration encompasses the system-level and algorithmic optimizations required to render and train 3D Gaussian Splatting (3DGS) efficiently on modern GPU architectures. It solves the bottlenecks of memory bandwidth, sorting latency, and GPU memory capacity that arise when processing scenes with tens of millions of primitives. It sets itself apart from sibling methods by focusing on **system-level distribution** (sharding across nodes), **hardware-specific kernel design** (Tensor Cores, Roofline optimization), and **memory hierarchy management** (CPU-GPU offloading) rather than just intra-GPU rasterization logic like tiling or sorting.

### [Occupancy Region Derivation & ϵ-Intersection Criterion](19020-occupancy-region-derivation-intersection-criterion.md)
This method defines a robust geometric intersection test between 3D Gaussian splats by deriving an elliptic occupancy region $\Omega_i$ from the Gaussian kernel's covariance and opacity, rather than relying on strict binary overlap of infinite support. It solves the problem of robustly determining whether two splats $G_i$ and $G_j$ interact for culling or sorting purposes, avoiding false negatives from minor misalignments or numerical precision issues. It differs from standard bounding-box culling by using a shape-aware elliptic boundary and an $\epsilon$-tolerance for contact detection, providing a balance between geometric accuracy and computational efficiency.

### [Gaussian Piling](19322-gaussian-piling.md)
Gaussian Piling is a slice-based volumetric reconstruction method that unifies 3D Gaussian Splatting with an imaging system-aware focus model to compress aggressively while preserving internal structure. Unlike standard 3D Gaussian Splatting, which discards internal volumetric information and produces incoherent 3D structures with floating artifacts for slice-based modalities, Gaussian Piling models the finite focal zone to preserve high-frequency internal volumetric detail. It achieves this by enforcing 3D-to-2D consistency through axial re-parameterization and opacity modulation, preventing ghosting artifacts across slices.

### [Parallel Differentiable Rasterization](19784-parallel-differentiable-rasterization.md)
Parallel Differentiable Rasterization is a rendering method that jointly produces High Dynamic Range (HDR) and Low Dynamic Range (LDR) images from a set of 3D Gaussians using two parallel rasterization pipelines, $F_{HDR}$ and $F_{LDR}$. It solves the problem of simultaneous HDR/LDR view synthesis by decoupling the color representation (HDR vs. LDR) while sharing the geometric projection and alpha-blending logic. It differs from standard differentiable rasterization by explicitly maintaining two distinct color streams for the same geometric primitives to handle wide dynamic range scenes without tone-mapping artifacts during training.

### [Local-GS](23282-local-gs.md)
Local-GS is a warp-coherent rendering paradigm that restructures alpha computation and reorganizes work around SIMT (Single Instruction, Multiple Threads) execution boundaries to reduce warp divergence and redundant computation in 3D Gaussian Splatting. Unlike methods that focus on macroscopic scene culling or data compression, Local-GS operates as a pure kernel-level optimization that shifts focus to microscopic GPU execution boundaries, mitigating intra-block computational overhead. It achieves this by enforcing tile-local execution through three stages: parameter hoisting, warp-level culling, and coherent blending.

### [CoherentRaster](23405-coherentraster.md)
CoherentRaster is a real-time 3D Gaussian Splatting (3DGS) rendering framework optimized for Light Field Displays (LFDs). It solves the computational bottleneck of dense multi-view synthesis by leveraging subpixel-level rasterization to eliminate redundant computations across adjacent viewpoints and resolve uncoalesced memory access patterns inherent to subpixel rendering. Unlike Subpixel-3DGS, which suffers from redundant cross-view computation and irregular memory access, or MPI-based frameworks, which scale poorly with resolution due to depth plane traversal, CoherentRaster achieves real-time performance on high-resolution scenes by exploiting view coherence.

### Conditional Splat Approximation
#### definition
- [S1, unofficial] A technique to approximate the influence range of a 3D Gaussian on a slice by using a 2D ellipse splat scaled by the marginal factor, which closely matches the true iso-probability contours.
#### number
- [S1, unofficial] Qualitative validation shows that the conditional splat approximation (dashed red contours) closely overlaps with the true 3D Gaussian iso-probability contours (solid blue contours).
#### comparison
- [S1, unofficial] The conditional splat provides an accurate and efficient approximation of the original 3D Gaussian’s influence range.

### Conservative Region Pre-Culling
#### definition
- [S2, unofficial] A pre-rasterization optimization that skips spatially contiguous blocks of Gaussians before projection and preprocessing, reducing enumeration, sorting, and downstream rasterization overhead.
#### step
- [S2, unofficial] At export time, reorder the Gaussian model by Morton code and cut it into fixed-size contiguous memory blocks.
- [S2, unofficial] For each block, precompute an axis-aligned bounding box that contains the full $\tau_0$-support of every Gaussian it holds, inflated by a safety factor.
- [S2, unofficial] Pass surviving blocks to the rasterizer as contiguous ranges; Gaussians in rejected blocks are never read, incurring zero cost in projection, prefix scan, or duplication.
#### parameter
- [S2, unofficial] $\tau_0$: The finite-support radius used by the backbone's Gaussian culling predicate.
#### layout
- [S2, unofficial] Gaussians are stored in contiguous memory blocks ordered by Morton code. Each block has a precomputed AABB. Rejected blocks are passed to the rasterizer as contiguous ranges to be skipped.
#### pitfall
- [S2, unofficial] Conservative culling may skip visible Gaussians if their support extends beyond the inflated block box. Handled by inflating the box with a safety factor to guarantee bit-exactness.
#### number
- [S2, unofficial] Evaluated across 52 runs; images produced are bit-identical to the unmodified backbone.

### GaussianPile Forward Rendering
#### definition
- [S3, unofficial] A forward rendering process for slice-based volumetric reconstruction that converts 3D Gaussian primitives into 2D images by convolving the Gaussian with a reversed sensitivity map (PSF) along the axial direction, resulting in a 'focus Gaussian' that accounts for depth-of-field effects.
#### step
- [S3, unofficial] Represent each primitive as an anisotropic 3D Gaussian with parameters $G = (\mu, s, q, \alpha)$, where $\mu$ is position, $s$ is scale, $q$ is orientation quaternion, and $\alpha$ is opacity.
- [S3, unofficial] Convert scale $s$ and orientation $q$ into a 3D covariance matrix $\Sigma$ in world coordinates: $S = \text{diag}(\text{mod} \cdot s)$, $R = \text{quatToMatrix}(q)$, $\Sigma = R S S^\top R^\top$.
- [S3, unofficial] Extract the 2D components from the focus Gaussian: $\Sigma_{2d}$ is the top-left 2x2 block of $\Sigma_e$, and $\mu_{2d} = [\mu_e[0], \mu_e[1]]^\top$.

### GES Implementation and Rendering Pipeline
#### parameter
- [S4] $\delta$: Depth offset in Eq. (2) and (3). Set $\delta = \sum_{k=1}^D s^k$ for each Gaussian, where $D$ is dimension of scaling and $s^k$ is the $k$-th axis length of scaling $\mathbf{s}$. This allows $\delta$ to vary adaptively to geometric granularity.
#### layout
- [S4] Optimization renderer: Pytorch framework, per-pixel rendering (GPU thread per pixel querying surfels and Gaussians).
- [S4] Post-optimization renderer: OpenGL, per-primitive rendering (GPU thread per fragment of primitives querying covered pixels). Uses 4x MSAA instead of 4x supersampling. Two rendering passes: 1. Depth testing/writing enabled, blending disabled. Geometry shader generates surfels from points, renders surfel color map and depth map. Multisampling textures resolved. Depth map modified by offset $\delta$ via post-processing shader. 2. Depth testing and additive blending enabled, depth writing disabled. Geometry shader generates 3D Gaussians from points, splats 3D Gaussians to obtain $\mathbf{C}$ and $\mathbf{D}$. Final image computed using Eq. (5) via post-processing shader.
#### pitfall
- [S4] Depth offset $\delta$ impact: If too large, 3D Gaussians occluded by surface leak out. If too small, 3D Gaussians close to surface are wrongly truncated.

## Background
The rendering pipeline for 3D Gaussian Splatting was introduced by Kerbl et al. in "A Real-Time Cyclic Rendering of 3D Gaussian Splatting" (2023) [S1, unofficial]. It builds on the foundational work of 3D Gaussian Splatting [G]. The method has since been adapted for various specialized domains including slice-based volumetric reconstruction [S3, unofficial], light field displays [S1, unofficial], and high-fidelity radiance field rendering [S4].

## Sources
[S1] InnerGS: Internal Scenes Reconstruction and Segmentation via Factorized 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2508.13287, fetched 2026-10-26 (unofficial)
[S2] TileSkipper: Region-Adaptive Tile Pruning for 3D Gaussian Splatting, https://arxiv.org/abs/2610.09343, fetched 2026-10-29 (unofficial)
[S3] GaussianPile: A Unified Sparse Gaussian Splatting Framework for Slice-based Volumetric Reconstruction, https://doi.org/10.48550/arXiv.2603.20611, fetched 2026-10-27 (unofficial)
[S4] When Gaussian Meets Surfel: Ultra-fast High-fidelity Radiance Field Rendering, https://doi.org/10.1145/3730925, fetched 2026-10-26
