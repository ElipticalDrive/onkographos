# Hardware-Accelerated and GPU Implementations

Also called: GPU Volume Ray Casting, Hardware-Accelerated DVR

## Overview
This family encompasses methods that offload the computationally intensive stages of direct volume ray casting—specifically sampling, interpolation, and compositing—to the graphics processing unit (GPU) [5616-hardware-accelerated-volume-rendering.md]. It solves the performance bottleneck of software-based ray casting, enabling real-time or near-real-time visualization of large 3D scalar datasets [5616-hardware-accelerated-volume-rendering.md]. It is distinguished from CPU-based implementations by leveraging dedicated texture units for hardware interpolation and fragment shaders for parallel compositing [5616-hardware-accelerated-volume-rendering.md], and from neural methods by using explicit voxel grids [1576-neural-volume-rendering.md].

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

### [Direct Volume Rendering](1184-direct-volume-rendering.md)
The foundational image-order algorithm treating the volume as a translucent medium. It is distinguished by its specific reliance on the ray-casting image-order algorithm and emission-absorption model, serving as the logical basis for GPU implementations [1184-direct-volume-rendering.md].

### [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-volume-rendering.md)
Implements DVR by offloading sampling and compositing to GPU texture mapping hardware and fragment shaders. It differs from general DVR by explicitly leveraging graphics APIs (OpenGL/Vulkan) and dedicated texture units for trilinear interpolation [5616-hardware-accelerated-volume-rendering.md].

### [Volume Ray-Casting on Graphics Hardware](7443-volume-ray-casting-on-graphics-hardware.md)
A multi-pass approach for DirectX 9-level hardware that exploits intrinsic parallelism via a stream model. It sets itself apart by integrating early ray termination and empty-space skipping directly into the fragment pipeline using early z-tests driven by depth replacement to discard unnecessary processing [7443-volume-ray-casting-on-graphics-hardware.md].

## Background
The shift to GPU acceleration began as CPU-based ray casting became insufficient for interactive frame rates with large datasets. Early implementations focused on utilizing the fixed-function pipeline and later programmable shaders to parallelize the per-pixel ray marching loop. The use of 3D textures for hardware interpolation was a critical development, reducing the computational load on the shader cores [5616-hardware-accelerated-volume-rendering.md].

## Sources
[S1] Volume Ray-Casting on Graphics Hardware, https://www.researchgate.net/publication/220668888_Volume_Ray-Casting_on_Graphics_Hardware, 2024-05-20
