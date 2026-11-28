# Interpolation and Sampling Techniques

Also called: Trilinear Interpolation (DRR), Trilinear rendering

## Overview
Interpolation and Sampling Techniques approximate continuous scalar fields from discrete voxel data during ray casting, determining the trade-off between computational cost and visual smoothness [G]. This family is distinguished from [Anti-Aliasing and Filtering](1574-anti-aliasing-and-filtering.md) by focusing on spatial reconstruction of the volume function rather than post-processing, and from [Neural volume rendering](1576-neural-volume-rendering.md) by typically relying on explicit geometric kernels or lookup tables [G]. A prominent example is trilinear interpolation, which offers significant speed advantages over exact intersection methods like Siddon's method for low sample counts [S1, unofficial].

## Shared principles
All methods in this family approximate a continuous scalar field $V(\mathbf{p})$ from discrete samples $V_i$ at grid positions $\mathbf{p}_i$ [G]. The core mechanism involves:
1. **Query Positioning**: Determining the 3D coordinate $\mathbf{p}$ along the ray $r(t) = \mathbf{o} + t\mathbf{d}$ where a value is needed [G].
2. **Kernel/Weight Application**: Computing weights $w_i(\mathbf{p})$ based on the distance between $\mathbf{p}$ and sample centers $\mathbf{p}_i$ to calculate $V(\mathbf{p}) = \sum_i w_i(\mathbf{p}) V_i$ [G].
3. **Coordinate Transformation**: Mapping world-space or camera-space coordinates to the volume's local coordinate system before interpolation [G].

## Subtopics

### [Depth to Distance Conversion](2428-depth-to-distance-conversion.md)
Transforms sampled depth values ($z\_vals$) along a camera ray into physical segment lengths ($dists$) between consecutive samples. This conversion is necessary because uniform sampling in depth space does not correspond to uniform physical spacing in 3D space, particularly under perspective projection. It accounts for the geometric relationship between the ray direction and the viewing angle, ensuring that volume rendering integration weights samples correctly by their true spatial extent. Unlike [NDC Points to Distance](2500-ndc-points-to-distance.md), which operates in Normalized Device Coordinates, this method operates directly on camera-space depth values.

### [Sampling and Jittering](2516-sampling-and-jittering.md)
Samples voxel data with jittered step offsets to reduce banding, and composites the result using front-to-back alpha blending. It differs from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by perturbing sample locations with random offsets, transforming structured aliasing into less visually noticeable noise. It is particularly effective when using low sample counts or nearest-neighbor interpolation, where regular grid patterns are most visible.

### [Space Warping (Log+Warp)](5815-space-warping-log-warp.md)
Warps 3D space towards the view cell center using radial distortion to bring the background closer for positional encoding, dampening high frequencies in the background while preserving foreground detail. It sets itself apart from its parent [Interpolation and Sampling Techniques](2787-interpolation-and-sampling-techniques.md) by explicitly warping the 3D coordinate space before encoding, and from its sibling [NDC Warping](7267-ndc-warping.md) by using a radial distortion function rather than a linear NDC squash.

### [MLS Volume Reconstruction](9796-mls-volume-reconstruction.md)
Approximates a continuous scalar field and its gradients from discrete samples (regular or irregular) using Moving Least Squares (MLS) approximation. It solves the problem of high-quality visualization of arbitrary volumetric datasets by providing a unified framework for function reconstruction that supports adaptive preintegration. Unlike Radial Basis Functions (RBFs), MLS does not require solving a global system of equations, making it more efficient for large datasets. It sets itself apart from standard interpolation methods by allowing tunable smoothing and interpolation control through basis and weighting functions, and from standard ray casting by integrating adaptive subdivision based on preintegration tables to handle the computational cost of the scalar field evaluation.

### [Interpolation Methods](10526-interpolation-methods.md)
Approximates a continuous volumetric field $V(\mathbf{p})$ from discrete samples $V_i$ located at grid positions $\mathbf{p}_i$ to determine values along viewing rays. This family is distinguished from sibling methods like [Depth to Distance Conversion](2428-depth-to-distance-conversion.md) and [Sampling and Jittering](2516-sampling-and-jittering.md) by its exclusive focus on the spatial weighting and blending of data values, rather than coordinate mapping or noise reduction. It sets itself apart from [Kernel-based Resampling](16228-kernel-based-resampling.md) by encompassing a broader range of techniques including non-kernel approaches (e.g., nearest neighbor, gradient-based splatting) and specialized hardware-accelerated primitives.

### [Pre-filtered Cubic B-spline Interpolation](13450-pre-filtered-cubic-b-spline-interpolation.md)
A high-quality sampling technique for direct volume ray casting that improves reconstruction accuracy by compensating for the low-pass filtering effect of the cubic B-spline kernel. Unlike standard cubic B-spline interpolation, which can cause excessive blurring, this method pre-processes the voxel data to invert the kernel's frequency response, allowing for sharper results without increasing memory footprint. It is chosen when visual fidelity is prioritized over the raw performance of hardware-accelerated tri-linear interpolation.

### [Opacity-Weighted Trilinear Interpolation Volume Rendering](14920-opacity-weighted-trilinear-interpolation-volume-rendering.md)
A direct volume ray casting technique that improves visual accuracy by interpolating the product of opacity and material color, rather than interpolating color and opacity separately. This method calculates the material color at a resampling point using opacity weighting before interpolation and computes lighting directly on the resampled points using interpolated normals, thereby avoiding color distortion and Mach band artifacts common in methods that interpolate shaded voxel colors. It sets itself apart from standard trilinear interpolation by preventing "intermediate colors" at material interfaces where transparent media would otherwise inappropriately influence the final color.

### [Post-Interpolative Importance Function](15021-post-interpolative-importance-function.md)
A visibility optimization technique for Monte Carlo volume rendering that maps volume data to a discrete set of importance values after interpolation, rather than before. It solves the problem of accurately visualizing small, thin structures (e.g., bone periosteum) which are often lost or blurred by pre-interpolative importance functions due to smoothing artifacts. Unlike pre-interpolative methods that apply importance to continuous scalar values, this method creates a step function along the ray, preserving fine details and eliminating the need for smoothing discontinuities.

### [Sampling and Integration Schemes](15420-sampling-and-integration-schemes.md)
Defines the computational framework for evaluating the volume rendering equation along a viewing ray, converting discrete volumetric data into a final pixel color. This family encompasses methods that determine *where* to sample the volume (stochastic vs. deterministic spacing) and *how* to aggregate those samples (compositing, projection, or approximation). It is distinguished from [Interpolation and Sampling Techniques](2787-interpolation-and-sampling-techniques.md) by focusing on the integration logic and sampling distribution rather than the spatial interpolation kernel, and from [Ray Marching and Discretization](2788-ray-marching-and-discretization.md) by encompassing both the sampling strategy and the reduction phase.

### Space-covering volume sampling
A technique to extend pre-integrated volume rendering to multidimensional transfer functions [S2].

### Irradiance Volume Sampling
A grid of irradiance samples is taken throughout the scene; at render time, the volume is queried and near-by irradiance samples are interpolated to estimate the global illumination at a point [S3, unofficial].

## Background
The use of trilinear interpolation for volume rendering, particularly in the context of Digital Radiographic Reconstructions (DRR), has been highlighted for its significant speed advantage over exact intersection methods like Siddon's method [S1, unofficial]. For instance, with 25 sample points, trilinear rendering takes approximately 737 μs, whereas Siddon's method takes ~25 ms; the costs match only when the sample count reaches ~2,000 points [S1, unofficial].

## Sources

[S1] Trilinear rendering - diffdrr, https://vivekg.dev/DiffDRR/tutorials/trilinear.html, fetched 2026-10-28 (unofficial)
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S3] PDF Irradiance Volumes for Games, https://chrisoat.com/papers/Oat_GDC2005_IrradianceVolumesForGames.pdf, fetched 2026-10-28 (unofficial)
