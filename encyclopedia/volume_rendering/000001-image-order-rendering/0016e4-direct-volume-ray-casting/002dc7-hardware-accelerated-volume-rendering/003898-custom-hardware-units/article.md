# Custom Hardware Units
## Overview
This family implements direct volume ray casting using dedicated, non-standard hardware accelerators rather than general-purpose GPUs or CPUs. It solves the performance bottleneck of real-time volume visualization by offloading specific pipeline stages (geometry resampling or optical compositing) to fixed-function or specialized DSP units. It is distinguished from [Hardware-Accelerated Volume Rendering via 3D Texturing](2799-hardware-accelerated-volume-rendering-via-3d-texturing.md) by its reliance on custom silicon architectures (e.g., VIRIM) rather than standard graphics APIs and texture units.
## Shared principles
The family implements the emission-absorption optical model defined in [Direct volume ray casting](19-direct-volume-ray-casting.md). All methods in this family adhere to the following shared mechanisms:

1.  **Ray Generation via Geometry or Projection:** Rays are generated per-pixel. This is typically achieved by rendering the volume's bounding box geometry (front and back faces) to determine entry and exit points, or by computing ray parameters directly from camera matrices and projection models (e.g., MCOP) in the vertex/fragment shader [S1, unofficial][S2][S3, S4, unofficial].
2.  **Hardware-Accelerated Sampling:** Scalar volume values $V(\mathbf{p})$ are sampled using the GPU's texture hardware. This involves trilinear interpolation of the 3D texture memory, which is optimized for fetching adjacent voxels in a single memory access [S1, unofficial][S2][S3, S4, unofficial].
3.  **Fragment Shader Compositing:** The core ray marching loop, including classification (transfer function application) and front-to-back compositing, is executed within the fragment shader [S1, unofficial][S2][S3, S4, unofficial]. The standard compositing equation is applied:
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
4.  **Early Ray Termination (ERT):** To optimize performance, rays are terminated when the accumulated opacity $\alpha'$ exceeds a threshold (typically 0.95 or 1.0), as subsequent samples are fully occluded [S1, unofficial][S2][S3, S4, unofficial].

## Subtopics

### [Hardware-based Geometry Unit Resampling](3139-hardware-based-geometry-unit-resampling.md)
This method implements the geometry-intensive portion of the ray-casting pipeline using dedicated hardware accelerators, specifically within the VIRIM architecture. It solves the performance bottleneck of perspective calculations, gradient estimation, and volume data resampling, which typically consume the majority of computation time in software-based implementations. By offloading these operations to specialized hardware, it generates ray starting points and sample values for the subsequent ray-casting units. It is distinguished from general-purpose GPU ray casting by its fixed-function, object-order approach that replicates volume memory across parallel geometry units to ensure conflict-free access.

### [Hardware-based Direct Volume Ray Casting](5442-hardware-based-direct-volume-ray-casting.md)
This method implements direct volume ray casting using dedicated programmable Digital Signal Processors (DSPs) to handle the computationally intensive tasks of shading and compositing, decoupled from the geometry generation stage. It solves the bottleneck of real-time volume visualization by offloading the per-voxel light-material interaction calculations from a general-purpose CPU or generic graphics hardware to specialized hardware units. Unlike [Hardware-based Geometry Unit Resampling](3139-hardware-based-geometry-unit-resampling.md), which focuses on data transformation, this method focuses on the optical model execution, specifically supporting the Heidelberg Raytracing algorithm.

### Custom Hardware Rendering
#### definition
- [S1, unofficial] A direct volume rendering technique implemented via custom hardware.

## Background
Custom hardware volume rendering emerged as a distinct paradigm alongside raycasting, splatting, and texture mapping approaches in the early 2000s [S1, unofficial]. It was developed to address the computational limits of software-based rendering by utilizing specialized silicon, such as the VIRIM architecture, to achieve real-time performance for large volumetric datasets [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
