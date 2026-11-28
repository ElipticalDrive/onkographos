# Splat Kernels and Footprints
## Overview
Splat Kernels and Footprints define the geometric shape, spatial extent, and weighting function of the primitives used in splatting-based volume rendering [G]. They solve the problem of how to map 3D volumetric data (voxels, particles, or points) onto a 2D image plane while preserving visual fidelity, handling perspective distortion, and managing aliasing [G]. This family sets itself apart from basic splatting [Splatting Without the Blur](1157-splatting-without-the-blur.md) by introducing sophisticated mathematical models for the footprint function—such as elliptical Gaussian projections [EWA Splatting](1069-ewa-splatting.md) or polynomial splines [Polynomial and Spline Kernels](9995-polynomial-and-spline-kernels.md)—rather than relying solely on fixed-radius disks [G]. It differs from [Splat Extent and Projection](1792-splat-extent-and-projection.md) by focusing on the internal density distribution of the footprint rather than just its bounding boundaries [G].

## Shared principles
Builds on [Splatting](23-splatting.md).

## Subtopics

### [EWA Splatting](1069-ewa-splatting.md)
EWA Splatting is a high-quality volume rendering technique that eliminates aliasing artifacts by replacing the standard footprint function with a band-limited resampling filter. It solves the problem of visual artifacts caused by incorrect visibility determination and sampling rate changes in perspective projections, which occur when reconstruction kernels overlap or when the sampling rate falls below the Nyquist frequency. It sets itself apart from basic splatting by combining an elliptical Gaussian reconstruction kernel with a Gaussian low-pass filter to avoid aliasing, and from other variants by using a rigorous affine approximation of the projective mapping to derive an analytic 2D footprint.

### [Splatting Without the Blur](1157-splatting-without-the-blur.md)
Splatting Without the Blur is a volume rendering technique that performs classification and shading in screen space, after voxels have been projected onto the image plane, to avoid the edge blurring inherent in traditional pre-shaded splatting. Unlike [Pre-Classified Splatting](5410-pre-classified-splatting.md), which applies optical properties before projection and results in fuzzy edges due to kernel smoothing, this method preserves crisp object boundaries by deferring classification until after the density footprint is rasterized. It distinguishes itself from [Edge Preservation in Volume Rendering Using Splatting](6573-edge-preservation-in-volume-rendering-using-splatting.md) by manipulating the rendering pipeline stages rather than warping the reconstruction kernels themselves.

### [Splat Radius Adjustment Based on Normal Angle](1264-splat-radius-adjustment-based-on-normal-angle.md)
This method adjusts the radius of individual splats to prevent visual holes that occur when surface samples are elevated in the normal direction of their base plane. It sets the splat radius based on the angle between the splat's normal vector and the base plane's normal vector. Unlike fixed-radius splatting, which assumes a flat distribution, this approach increases radii in areas with high local elevation variance, ensuring continuous coverage. It builds on [Splatting](23-splatting.md) by modifying the primitive size parameter during the construction phase.

### [Splat Extent and Projection](1792-splat-extent-and-projection.md)
This family addresses the geometric problem of determining the screen-space position and spatial extent (bounding region) of a projected 3D kernel (splat) to optimize rasterization, culling, and clipping. It sets itself apart from sibling methods like [EWA Splatting](1069-ewa-splatting.md) by focusing specifically on the analytical or approximate derivation of the splat's footprint boundaries rather than the density evaluation or compositing order [G].

### [EWA Splatting with kNN Covariance Estimation](4690-ewa-splatting-with-knn-covariance-estimation.md)
A rendering method that uses per-point 3D covariance estimated from k-nearest neighbours to project an elliptical weighted average (EWA) shape onto the 2D image plane for denser coverage than simple disk splatting. It distinguishes itself from fixed-radius EWA splatting by adapting the kernel footprint to local point density, reducing holes in sparse regions and preventing excessive overlap in dense regions. It builds on [EWA Splatting](1069-ewa-splatting.md) by providing a data-driven covariance estimation step prior to the standard EWA projection.

### [Primitive Geometry](6264-primitive-geometry.md)
Primitive Geometry is the foundational family of splatting methods that decompose volumetric data into explicit geometric or parametric primitives (such as voxels, tetrahedra, or line segments) which are transformed and rasterized in screen space [G]. It differs from ray-casting by computing contributions per primitive rather than per pixel, leveraging hardware-accelerated rasterization for real-time performance [G].

### [Edge-Preserving Splatting with Compression/Stretching](6574-edge-preserving-splatting-with-compression-stretching.md)
Edge-Preserving Splatting is a volume rendering technique that sharpens discontinuities in low-resolution or undersampled data by applying non-linear compression and stretching functions to standard splatting kernels. Unlike conventional splatting, which produces smooth, blurred edges, this method ensures C1 continuity (continuous first derivative) across splat boundaries while enhancing the visual definition of iso-surfaces. It distinguishes itself from sibling methods like [EWA Splatting](1069-ewa-splatting.md) by explicitly modeling edge geometry (direction and strength) to drive view-dependent footprint selection, rather than relying solely on geometric projection.

### [Polynomial and Spline Kernels](9995-polynomial-and-spline-kernels.md)
Polynomial and Spline Kernels refer to a family of splatting methods that replace standard Gaussian reconstruction kernels with piecewise polynomial, spline, or other non-Gaussian functions to control the shape, support, and decay of the splat footprint. They solve the problem of visual artifacts, blurring, or inflexible footprint shapes inherent in Gaussian kernels by providing kernels with compact support, tunable parameters, or specific decay properties that better preserve high-frequency features or enable differentiable optimization. They set themselves apart from siblings like [EWA Splatting](1069-ewa-splatting.md) by focusing on the mathematical form of the kernel function itself rather than just the projection geometry, and from [Splat Extent and Projection](1792-splat-extent-and-projection.md) by defining the internal weight distribution within the footprint.

### Embedded Vector Icons in Splat Footprint
- **Definition:** A refinement where vector icons are embedded directly into the splat footprint as billboard images overlapped and composited [S1, unofficial].
- **Step:** Overlap and composite small billboard images to build up the final image [S1, unofficial]. Place a small icon within the billboard image [S1, unofficial]. Orient the billboard image perpendicular to the viewing ray and parallel to the projected vector direction at the splat’s center point [S1, unofficial]. Use a discrete set of billboard images with vector icons reduced in size [S1, unofficial].

### Falloff Penalty Prior
- **Definition:** A regularization term that flattens the kernel falloff to bound each primitive's blending footprint, reducing overdraw and primitive count [S2, unofficial].
- **Formula:** Prior term added to loss: $\lambda L_\beta$ [S2, unofficial].
- **Parameter:** $\lambda$: falloff penalty strength. Values tested: 0 (off), 0.005 (optimal), 0.01, 0.05 [S2, unofficial].
- **Pitfall:** High $\lambda$ (e.g., 0.05) creates fully hard-edged discs, forcing densification to add primitives to cover holes, increasing count and dropping PSNR [S2, unofficial].
- **Number:** At $\lambda=0.005$, hardware frame rate rises 1.46–1.69× for 0.001–0.021 LPIPS penalty. At $\lambda=0.05$, discs become fully hard-edged, densification adds surfels to cover holes, and PSNR drops 2.1 dB [S2, unofficial].
- **Comparison:** Without texture, the prior flattens color interpolation between primitives, costing 0.53 dB quality loss versus 0.33 dB with texture for similar overdraw reduction [S2, unofficial].
- **Usage:** Applied during teacher training phase of the baking pipeline [S2, unofficial].

### Textured Splat Rendering
- **Definition:** Rendering each point as an oriented quad where the fragment shader computes per-fragment texture coordinates to retrieve color and normal data, allowing high-frequency detail recovery with fewer points [S3].
- **Step:** For each point $p$ in the cloud, render an oriented quad [S3]. In the fragment shader, receive the interpolated position of the fragment and the point's scan location $s_k$ [S3]. Compute per-fragment texture coordinates using the inverse mapping equations based on the vector from $s_k$ to the fragment's position [S3]. Retrieve fragment properties (color, normals) from the texture map using the computed texture coordinates [S3]. Use the alpha channel of the color texture to discard fragments outside the silhouette of the model [S3]. Generate circular or ellipsoidal splats based on the distance to the point center [S3].
- **Parameter:** Geometry storage: Regular $(x,y,z)$ data for point coordinates to avoid precision loss from 16-bit depth encoding [S3]. Color storage: Texture maps (potentially lossy compressed) to reduce disk memory footprint [S3].
- **Pitfall:** Regular scaled splats yield poor results because properties (color, normal) are constant across the splat surface, losing high-frequency detail [S3].

## Background
The concept of splatting originated in the early 1990s as a method for efficient volume rendering [G]. The term "splat" refers to the shape of the footprint projected onto the screen [G]. Early implementations used simple Gaussian kernels [G]. The development of Elliptical Weighted Average (EWA) splatting addressed the aliasing issues inherent in perspective projections by deriving analytic 2D footprints [G]. Subsequent research introduced polynomial kernels [S2, unofficial][S3], edge-preserving techniques [S1, unofficial], and noise-based splats [S1, unofficial] to handle specific visual artifacts and data types. More recently, splatting has been adapted for neural rendering and 3D Gaussian Splatting, where covariance matrices and advanced weighting schemes are used to optimize rendering quality and speed [G].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
