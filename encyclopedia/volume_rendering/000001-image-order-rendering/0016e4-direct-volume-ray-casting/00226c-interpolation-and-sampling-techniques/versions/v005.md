# Interpolation and Sampling Techniques

Also called: Trilinear Interpolation (DRR), Trilinear rendering

## Overview
Interpolation and Sampling Techniques define how continuous scalar fields are reconstructed from discrete voxel data to evaluate density or opacity at arbitrary query positions along viewing rays [10526-interpolation-methods.md]. This family solves the problem of discretization artifacts (aliasing, blockiness) inherent in nearest-neighbor sampling while balancing computational cost against spatial smoothness [S2, unofficial][S3]. It sets itself apart from sibling techniques like [Supersampling](3493-supersampling.md) (which increases sample count) or [Multi-Sample Anti-Aliasing via Ray Distribution](1219-multi-sample-anti-aliasing-via-ray-distribution.md) (which distributes rays) by focusing on the mathematical reconstruction of the field value itself from existing samples [10526-interpolation-methods.md].

## Shared principles
All methods in this family map a query position $\mathbf{p}$ in voxel space to a scalar value $V(\mathbf{p})$ by combining a finite set of neighboring voxel values $V_i$ using a set of weights $w_i(\mathbf{p})$:
$$
V(\mathbf{p}) = \sum_{i} w_i(\mathbf{p}) V_i
$$
The weights are derived from a reconstruction kernel $h(\mathbf{z})$, where $\mathbf{z}$ is the distance from the query point to the sample center [G]. The choice of kernel defines the trade-off between computational complexity, spatial extent (number of neighbors), and frequency response (aliasing vs. blurring) [S1, unofficial][S2, unofficial].

## Subtopics

### [Multi-Sample Anti-Aliasing via Ray Distribution](1219-multi-sample-anti-aliasing-via-ray-distribution.md)
Multi-Sample Anti-Aliasing via Ray Distribution reduces jagged edges and aliasing artifacts in volume rendering by casting multiple rays per pixel, distributed within the pixel area, and averaging the results. It sets itself apart from standard single-ray casting by explicitly addressing the coarse granularity of per-pixel sampling, which causes abrupt intensity changes along object silhouettes. Compared to [Supersampling](3493-supersampling.md), which generally implies a uniform grid increase in resolution, this method employs specific deterministic offset patterns to distribute samples efficiently within the pixel footprint.

### [Depth to Distance Conversion](2428-depth-to-distance-conversion.md)
Depth to Distance Conversion transforms discrete depth samples ($z\_vals$) along a ray into physical segment lengths ($dists$) required for volume rendering integration. It solves the problem that raw depth differences do not represent true Euclidean distances when the ray is not aligned with the viewing direction. It differs from [NDC Points to Distance](2500-ndc-points-to-distance.md) by operating in camera-space depth coordinates rather than Normalized Device Coordinates, making it suitable for standard perspective projection pipelines where depth is linear in camera space but non-linear in screen space.

### [Alpha Grid Mask](2430-alpha-grid-mask.md)
The Alpha Grid Mask is a module that stores a 3D alpha volume and provides efficient sampling of alpha values at arbitrary 3D coordinates using trilinear interpolation. It serves as the core sampling primitive for volume rendering to incorporate spatially varying opacity constraints derived from external supervision or geometry masks.

### [Sampling and Jittering](2516-sampling-and-jittering.md)
Sampling and Jittering is a direct volume rendering technique that casts rays from the camera through a 3D voxel grid, samples voxel data with jittered step offsets to reduce banding artifacts, and composites the result using front-to-back alpha blending. It perturbs sample positions along each ray using a pre-computed jitter texture to break up regular grid patterns.

### [Slab-based Ray Sampling](2853-slab-based-ray-sampling.md)
Slab-based ray sampling is an image-order caching mechanism for direct volume ray casting that groups rays into rectangular screen regions to build stacked slabs reaching into the scene. Unlike object-order bricking, which relies on per-brick lists inefficient for GPU hardware, this method stores voxel sampling results in a regular three-dimensional array in shared memory, optimizing memory bandwidth and latency for lighting techniques that require neighborhood access, such as gradient calculation or ambient occlusion.

### [Supersampling](3493-supersampling.md)
Supersampling is a technique to minimize aliasing artifacts by casting multiple rays per pixel and averaging the results, effectively sampling the scene at a higher spatial frequency than the displayed resolution. It addresses the coarse granularity of per-pixel sampling, which causes visible artifacts such as jagged edges (staircase artifacts) and moiré patterns, particularly along object silhouettes where intensity changes abruptly. Unlike standard single-sampling, supersampling provides a more accurate color representation for each pixel by integrating the signal over the pixel area.

### [Epipolar Sampling](3899-epipolar-sampling.md)
Epipolar sampling is a technique for aggregating features from multiple views by sampling along epipolar lines rather than along camera rays in 3D space. It solves the problem of efficiently correlating information between a reference view and target views without performing full 3D volume rendering or explicit 3D reconstruction during the encoding phase. Unlike standard volume rendering which samples along a ray, this method samples features from a reference pixel and correlates them with features sampled along the corresponding epipolar line in a second view, augmented with positionally encoded depths. This allows the encoder to memorize the correct depth consistent with the scene's arbitrary scale, which is then used for subsequent parameter prediction.

### [Hierarchical Sampling](4865-hierarchical-sampling.md)
Hierarchical sampling is a two-stage sampling strategy used in neural volume rendering (specifically NeRF) to improve rendering quality and efficiency by concentrating samples in regions of high geometric or radiance complexity. It solves the problem of inefficient uniform sampling, where many samples fall in empty space or smooth regions, by using a coarse network to estimate the density distribution and then importance-sampling along the ray based on that estimate. It sets itself apart from standard direct volume ray casting [19-direct-volume-ray-casting.md] by replacing uniform discretization with a learned, non-uniform probability density function (PDF) derived from a separate coarse model, and from [Neural volume rendering](1576-neural-volume-rendering.md) by explicitly decoupling the density estimation (coarse) from the final color prediction (fine) into two distinct passes.

### [Space Warping (Log+Warp)](5815-space-warping-log-warp.md)
Space Warping (Log+Warp) is a positional encoding strategy for neural radiance fields that warps 3D space towards the view cell center using radial distortion. It solves the problem of high-frequency background details requiring excessive network capacity by dampening these frequencies while preserving foreground detail. It sets itself apart from standard logarithmic sampling by applying an additional inverse-square-root warp to the encoded coordinates, ensuring that sample locations in space remain consistent across different viewpoints.

### [MLS Volume Reconstruction](9796-mls-volume-reconstruction.md)
MLS Volume Reconstruction is a volume rendering technique that reconstructs a smooth scalar field from discrete samples (regular or irregular grids) using Moving Least Squares (MLS) approximation, allowing for adaptive preintegration to handle the computational cost of the scalar field evaluation. It solves the problem of high-quality visualization of arbitrary volumetric datasets by providing a unified mathematical framework for function reconstruction and shading, subsuming convolution as a special case. Unlike Radial Basis Functions (RBFs), MLS does not require solving a global system of equations, making it more efficient for large datasets.

### [Interpolation Methods](10526-interpolation-methods.md)
Interpolation methods reconstruct a continuous scalar field from discrete voxel data to evaluate density or opacity at arbitrary query positions along viewing rays [G]. This family solves the problem of discretization artifacts (aliasing, blockiness) inherent in nearest-neighbor sampling while balancing computational cost against spatial smoothness. It sets itself apart from sibling techniques like [Supersampling](3493-supersampling.md) (which increases sample count) or [Multi-Sample Anti-Aliasing via Ray Distribution](1219-multi-sample-anti-aliasing-via-ray-distribution.md) (which distributes rays) by focusing on the mathematical reconstruction of the field value itself from existing samples.

#### Svox1 Rendering Formula (Nearest-Neighbor)
Implements the Svox1 rendering formula using nearest-neighbor interpolation with infinitely many steps. Includes forward, backward, and fused variants [S2, unofficial].

#### Space-covering volume sampling
A technique to extend pre-integrated volume rendering to multidimensional transfer functions [S3].

## Background
Trilinear interpolation is widely used in Digital Radiographic Reconstruction (DRR) for its speed advantage over exact intersection methods like Siddon's method [S1, unofficial]. In a benchmark using the `diffdrr` library, Siddon's method took approximately 25 ms for a standard AP view, while trilinear interpolation with 25 samples took 737 μs. The cost of trilinear interpolation matches Siddon's method only when the number of samples per ray ($n\_points$) reaches approximately 2,000 (25 ms) [S1, unofficial]. Performance scales linearly with $n\_points$: 50 samples take 1.03 ms, 100 samples take 1.65 ms, and 5,000 samples take 66 ms [S1, unofficial].

## Sources

[S1] Trilinear rendering - diffdrr, https://vivekg.dev/DiffDRR/tutorials/trilinear.html, fetched 2026-10-28 (unofficial)
[S2] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S3] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
