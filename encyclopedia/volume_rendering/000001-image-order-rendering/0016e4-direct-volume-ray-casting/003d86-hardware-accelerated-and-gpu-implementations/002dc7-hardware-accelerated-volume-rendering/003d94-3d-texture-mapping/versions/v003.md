# 3D Texture Mapping

Also called: 3D-Texture Volume Rendering, Rendering Slices with 3D Texture Mapping, Voltx Method, Volume Rendering with Shading via Three-Dimensional Texture Maps, Volume Texturing

## Overview
3D Texture Mapping renders a volumetric dataset by storing it as a 3D texture in GPU memory and resampling it using hardware-accelerated trilinear interpolation on planar polygons [S1, unofficial][S6, unofficial]. It solves the performance bottleneck of software ray casting by offloading the expensive interpolation and sampling steps to the graphics hardware's texture engine [S6, unofficial]. Unlike [Shear-Warp](shear-warp.md), which requires a final warp step and performs classification before resampling, this method performs trilinear interpolation followed by classification and compositing, resulting in higher image quality and consistent performance for perspective projections [S1, unofficial].

## Shared principles
All methods in this family implement the emission-absorption optical model defined in [Direct volume ray casting](19-direct-volume-ray-casting.md). The core pipeline for each pixel consists of ray generation, hardware-accelerated sampling, classification, and compositing [5616-hardware-accelerated-volume-rendering.md].

1.  **Ray Generation**: Rays are generated per-pixel, typically by computing ray parameters from camera matrices in vertex/fragment shaders or by rendering the volume's bounding box geometry to determine entry/exit points [5616-hardware-accelerated-volume-rendering.md].
2.  **Hardware-Accelerated Sampling**: Scalar volume values are sampled using the GPU's texture hardware. The volume data is stored in 3D texture memory, and the texture engine performs trilinear interpolation of the eight surrounding voxels in parallel, avoiding manual shader interpolation [5616-hardware-accelerated-volume-rendering.md].
3.  **Fragment Shader Compositing**: The ray marching loop, including classification and front-to-back compositing, is executed within the fragment shader [5616-hardware-accelerated-volume-rendering.md]. The standard compositing equation updates accumulated color $C'$ and opacity $\alpha'$:
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
4.  **Early Ray Termination (ERT)**: Rays are terminated when accumulated opacity exceeds a threshold (typically 0.95 or 1.0) to optimize performance [5616-hardware-accelerated-volume-rendering.md].

## Subtopics

### [Hardware-Accelerated Volume Rendering via 3D Texturing](2799-hardware-accelerated-volume-rendering-via-3d-texturing.md)
This method renders a volumetric dataset by storing it as a 3D texture in GPU memory and resampling it using hardware-accelerated trilinear interpolation on planar polygons. It solves the performance bottleneck of software ray casting by offloading the expensive interpolation and sampling steps to the graphics hardware's texture engine. Unlike [Shear-Warp](shear-warp.md), which requires a final warp step and performs classification before resampling, this method performs trilinear interpolation followed by classification and compositing, resulting in higher image quality and consistent performance for perspective projections. It builds on [3D Texture Mapping](9529-3d-texture-mapping.md) by utilizing the hardware's ability to sample 3D texture coordinates on arbitrary polygons.

### [3D Pixel Texture Classification](6055-3d-pixel-texture-classification.md)
3D Pixel Texture Classification is a hardware-accelerated method for applying multi-dimensional transfer functions during direct volume ray casting by encoding classification parameters into a 3D texture lookup. It solves the problem of high-dimensional transfer function evaluation by mapping scalar volume properties (Data Value, Gradient Magnitude, and Second Derivative) to color and opacity via a single texture fetch. It differs from standard 1D or 2D lookup tables by explicitly incorporating boundary information (via the second derivative) into the hardware interpolation pipeline, allowing for precise control over material boundaries without requiring complex shader logic.

### [Hardware Shading via Pixel Texture](6056-hardware-shading-via-pixel-texture.md)
Hardware Shading via Pixel Texture is a technique that integrates surface shading into texture-memory based volume rendering by encoding quantized surface normals into a 2D pixel texture. It solves the problem of missing or low-quality shading in standard hardware-accelerated volume rendering, which often lacks the geometric information required for realistic lighting. Unlike standard 3D texture mapping which relies on hardware interpolation of scalar values, this method uses nearest-neighbor interpolation on a 2D texture to avoid the redundancy and truncation errors associated with interpolating normals in hardware. It sets itself apart from siblings like [3D Pixel Texture Classification](6055-3d-pixel-texture-classification.md) by specifically targeting the shading component via a separate pixel texture lookup rather than integrating shading into the primary classification volume.

### Texture Mapping Volume Rendering Optimization
Reducing texture-mapped polygons is equivalent to reducing the number of samples taken along the ray in an image-order ray casting method [S2, book]. Loading a reduced resolution volume is similar to reducing both the number of rays cast and the number of samples taken along a ray in an image-order method [S2, book]. To achieve faster rendering speeds, the number of texture-mapped polygons used to represent the volume can be reduced [S2, book]. If texture download rates are a bottleneck, a reduced resolution version of the volume can be loaded into texture memory for interaction [S2, book].

## Background
The method was introduced by Kindlmann et al. in 1996 [S1, unofficial]. It was developed to leverage the emerging capabilities of 3D texture hardware in graphics accelerators, such as the SGI Reality Engine, which supported 8-way interleaved memory access for efficient fetching of trilinear cells [S1, unofficial]. The technique became a standard approach for hardware-accelerated volume rendering in the late 1990s and early 2000s, particularly in medical imaging and scientific visualization applications where real-time or near-real-time performance was required [S1, unofficial][S2, book].

## Sources
[S1] Kindlmann, G. L., et al., "Volume Rendering with Shading via Three-Dimensional Texture Maps", https://ntrs.nasa.gov/api/citations/19970021689/downloads/19970021689.pdf?attachment=true, fetched 2026-10-29 (unofficial)
[S2] "Chapter 7 - Advanced Computer Graphics - VTK", https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
