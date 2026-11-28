# Interpolation and Sampling Techniques

Also called: Trilinear Interpolation (DRR), Trilinear rendering

## Overview
Interpolation and Sampling Techniques define how continuous scalar fields are reconstructed from discrete voxel data during the sampling phase of direct volume ray casting. They solve the problem of retrieving volume values $V(\mathbf{p})$ at arbitrary ray positions $\mathbf{p}$ that do not align with voxel grid centers. This family is distinguished from [Nearest Neighbor Sampling](11694-nearest-neighbor-sampling.md) by providing spatial continuity, and from [Pre-integration and Approximation Methods](12352-pre-integration-and-approximation-methods.md) by typically evaluating the transfer function per-sample rather than pre-computing integrals over segments, although it serves as the foundational sampling primitive for those methods.

## Shared principles
All methods in this family map a query position $\mathbf{p}$ in voxel space to a scalar value $V(\mathbf{p})$ by combining a finite set of neighboring voxel values $V_i$ using a set of weights $w_i(\mathbf{p})$:
$$
V(\mathbf{p}) = \sum_{i} w_i(\mathbf{p}) V_i
$$
The weights are derived from a reconstruction kernel $h(\mathbf{z})$, where $\mathbf{z}$ is the distance from the query point to the sample center [G]. The choice of scheme defines the trade-off between computational complexity, spatial extent (number of neighbors), and frequency response (aliasing vs. blurring) [S1, unofficial][S2].

## Subtopics

### [Depth to Distance Conversion](2428-depth-to-distance-conversion.md)
Transforms discrete depth samples into physical segment lengths for integration.

### [Alpha Grid Mask](2430-alpha-grid-mask.md)
Stores and samples a 3D alpha volume for spatially varying opacity constraints.

### [Sampling and Jittering](2516-sampling-and-jittering.md)
Perturbs sample positions to reduce banding and aliasing artifacts.

### [Space Warping (Log+Warp)](5815-space-warping-log-warp.md)
Warps 3D space to dampen high-frequency background details in neural rendering.

### [MLS Volume Reconstruction](9796-mls-volume-reconstruction.md)
Reconstructs smooth fields using Moving Least Squares for adaptive preintegration.

### [Interpolation Methods](10526-interpolation-methods.md)
General reconstruction of continuous fields from discrete voxels.

### Space-covering volume sampling
A technique to extend pre-integrated volume rendering to multidimensional transfer functions [S2].

### Pre-filtered Cubic B-spline Interpolation
An on-the-fly interpolation approach combined with a pre-filtering step to perform more accurate interpolations without larger data storage [S3, unofficial]. It involves pre-processing all voxel values and performing a combination of local lookups during ray evaluation [S3, unofficial].

### Variable Sample Distance / Random Offset
A strategy to break the stripe pattern of onion ring artifacts by varying sample locations of rays in adjacent pixels [S3, unofficial]. This is achieved by using variable sample distances or a random offset at the beginning of each ray, resulting in a more 'dithered' image where errors are randomly distributed [S3, unofficial].

## Background
Trilinear interpolation is widely used in volume rendering due to its simplicity, hardware support on GPUs, and acceptable visual quality for high-resolution datasets [S3, unofficial]. In the context of Digital Radiographic Reconstructions (DRR), trilinear rendering offers significant speed advantages over exact intersection methods like Siddon's method. For example, with 25 sample points per ray, trilinear interpolation takes approximately 737 μs, whereas Siddon's method takes ~25 ms. The cost of trilinear interpolation matches Siddon's method only when the number of sample points reaches approximately 2,000 [S1, unofficial].

## Sources
[S1] Trilinear rendering - diffdrr, https://vivekg.dev/DiffDRR/tutorials/trilinear.html, fetched 2026-10-28 (unofficial)
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S3] Common Artifacts in Volume Rendering, https://doi.org/10.48550/arxiv.2109.13704, fetched 2026-10-26 (unofficial)
