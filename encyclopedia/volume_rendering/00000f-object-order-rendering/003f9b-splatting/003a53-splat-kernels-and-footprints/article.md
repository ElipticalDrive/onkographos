# Splat Kernels and Footprints

Also called: Splat Kernels, Footprint Functions, Reconstruction Kernels

## Overview
Splat Kernels and Footprints define the spatial weighting function used to project volumetric primitives onto the 2D image plane during splatting [S1, S3, S4, unofficial]. They solve the problem of reconstructing a continuous volume from discrete samples by determining how much each primitive contributes to a specific pixel based on its distance from the primitive's center [S2, unofficial][S6, unofficial][S10, abstract only, unofficial]. This family is distinguished from ray-casting by computing contributions per primitive rather than per pixel, leveraging hardware-accelerated rasterization for real-time performance [S1, S2, S3, unofficial]. It sets itself apart from [Splatting Without the Blur](1157-splatting-without-the-blur.md) by explicitly defining the shape and decay of the reconstruction function, and from [EWA Splatting](1069-ewa-splatting.md) by focusing on the algebraic form of the radial basis function rather than just the covariance-based footprint approximation [S10, abstract only, unofficial].

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

### [EWA Splatting](1069-ewa-splatting.md)
EWA Splatting is a high-quality volume rendering technique that eliminates aliasing artifacts by replacing the standard footprint function with a band-limited resampling filter. It solves the problem of visual artifacts caused by incorrect visibility determination and sampling rate changes in perspective projections, which occur when reconstruction kernels overlap or when the sampling rate falls below the Nyquist frequency. It sets itself apart from basic splatting [Splatting Without the Blur](1157-splatting-without-the-blur.md) by combining an elliptical Gaussian reconstruction kernel with a Gaussian low-pass filter, and from [Splat Radius Adjustment Based on Normal Angle](1264-splat-radius-adjustment-based-on-normal-angle.md) by using a rigorous affine approximation of the projective mapping to derive an analytic 2D footprint.

### [Splat Extent and Projection](1792-splat-extent-and-projection.md)
This family addresses the geometric problem of determining the screen-space position and spatial extent (bounding region) of a projected 3D kernel (splat) to optimize rasterization, culling, and clipping. It sets itself apart from sibling methods like [EWA Splatting](1069-ewa-splatting.md) by focusing specifically on the analytical or approximate derivation of the splat's footprint boundaries rather than the density evaluation or compositing order [G].

### [Primitive Geometry](6264-primitive-geometry.md)
Primitive Geometry is the foundational family of splatting methods that decompose volumetric data into explicit geometric or parametric primitives (such as voxels, tetrahedra, or line segments) which are transformed and rasterized in screen space. It differs from ray-casting by computing contributions per primitive rather than per pixel, leveraging hardware-accelerated rasterization for real-time performance.

### [Edge-Preserving Splatting with Compression/Stretching](6574-edge-preserving-splatting-with-compression-stretching.md)
Edge-Preserving Splatting with Compression/Stretching is a volume rendering technique that sharpens discontinuities in low-resolution or undersampled data by applying non-linear compression and stretching functions to the standard splatting kernel. Unlike conventional splatting, which produces smooth, blurred edges, this method ensures C1 continuity (continuous first derivative) across splat boundaries, allowing for the reconstruction of sharp features such as cavity walls and small protrusions. It distinguishes itself from sibling methods by explicitly modeling edge direction and strength to drive view-dependent footprint selection, rather than relying solely on binary classification or fixed kernel shapes.

### [Noise Splats](8547-noise-splats.md)
Noise Splats is a multivariate volume rendering technique that extends textured splatting by embedding controlled noise or discrete dots within the reconstruction texture. It solves the problem of representing nonhomogeneous materials (e.g., translucent granite) or synthesizing realistic clouds/fog by using the local amount of noise as a second independent scalar variable. Unlike standard textured splatting which uses anisotropic textures for vector fields, Noise Splats embeds noise directly into the scalar splat footprint, allowing the variance of the rendered image to encode additional data.

### [Polynomial and Spline Kernels](9995-polynomial-and-spline-kernels.md)
This family defines splatting kernels using piecewise polynomial functions and spline basis functions to approximate volumetric density. It addresses the need for kernels with compact support and specific smoothness properties (continuity of derivatives) that differ from the standard Gaussian or EWA splats. It sets itself apart from [EWA Splatting](1069-ewa-splatting.md) by focusing on the algebraic form of the radial basis function rather than just the covariance-based footprint approximation, and from [Splat Extent and Projection](1792-splat-extent-and-projection.md) by defining the internal weight decay function.

### Embedded Vector Icons in Splat Footprint
#### definition
- [S1, unofficial] A refinement where vector icons are embedded directly into the splat footprint as billboard images overlapped and composited.
#### step
- [S1, unofficial] Overlap and composite small billboard images to build up the final image.
- [S1, unofficial] Place a small icon within the billboard image.
- [S1, unofficial] Orient the billboard image perpendicular to the viewing ray and parallel to the projected vector direction at the splat’s center point.
- [S1, unofficial] Use a discrete set of billboard images with vector icons reduced in size.

### Falloff Penalty Prior
#### definition
- [S2, unofficial] A regularization term that flattens the kernel falloff to bound each primitive's blending footprint, reducing overdraw and primitive count.
#### formula
- [S2, unofficial] Prior term added to loss: $\lambda L_\beta$.
#### parameter
- [S2, unofficial] $\lambda$: falloff penalty strength. Values tested: 0 (off), 0.005 (optimal), 0.01, 0.05.
#### pitfall
- [S2, unofficial] High $\lambda$ (e.g., 0.05) creates fully hard-edged discs, forcing densification to add primitives to cover holes, increasing count and dropping PSNR.
#### number
- [S2, unofficial] At $\lambda=0.005$, hardware frame rate rises 1.46–1.69× for 0.001–0.021 LPIPS penalty. At $\lambda=0.05$, discs become fully hard-edged, densification adds surfels to cover holes, and PSNR drops 2.1 dB.
#### comparison
- [S2, unofficial] Without texture, the prior flattens color interpolation between primitives, costing 0.53 dB quality loss versus 0.33 dB with texture for similar overdraw reduction.
#### usage
- [S2, unofficial] Applied during teacher training phase of the baking pipeline.

## Background
The concept of splatting as a volume rendering technique was introduced by Levoy [S1, S3, S4, unofficial]. The specific formulation of EWA (Elliptical Weighted Average) splatting was developed by Westin et al. to address aliasing in perspective projections [S2, S8, unofficial]. The use of polynomial and spline kernels has been explored to provide compact support and specific continuity properties [S10, unofficial]. Noise splats were introduced to handle multivariate data representation [S1, unofficial]. Edge-preserving techniques were developed to address the blurring of discontinuities in undersampled data [S1, unofficial]. The falloff penalty prior is a recent addition in the context of 3D Gaussian Splatting optimization [S2, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
