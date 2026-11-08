# Custom Hardware Units
Also called: Dedicated Volume Rendering Hardware, ASIC/FPGA Volume Rendering, Fixed-Function Volume Rendering

## Overview
Custom Hardware Units implement direct volume rendering using application-specific integrated circuits (ASICs) or field-programmable gate arrays (FPGAs) rather than general-purpose GPUs or CPUs [S2, unofficial]. This family solves the problem of achieving real-time interactive frame rates for large, regular volumetric datasets during eras when general-purpose graphics hardware lacked sufficient throughput [S1, unofficial]. Unlike [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-volume-rendering.md) methods that rely on programmable fragment shaders, custom hardware units utilize fixed-function pipelines optimized for the highly regular computation patterns of volume rendering, such as shear-warp or ray-marching [S1, unofficial].

## Shared principles
All methods in this family offload the computationally intensive tasks of volume rendering to dedicated hardware accelerators, decoupling them from general-purpose processors. The core pipeline for each pixel consists of ray generation, hardware-accelerated sampling, classification, and compositing [5616-hardware-accelerated-volume-rendering.md].

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

### [Hardware-based Geometry Unit Resampling](3139-hardware-based-geometry-unit-resampling.md)
This method implements the geometry-intensive portion of the ray-casting pipeline using dedicated hardware accelerators, specifically within the VIRIM architecture. It solves the performance bottleneck of perspective calculations, gradient estimation, and volume data resampling, which typically consume the majority of computation time in software-based implementations. By offloading these operations to specialized hardware, it generates ray starting points and sample values for the subsequent ray-casting units. It is distinguished from general-purpose GPU ray casting by its fixed-function, object-order approach that replicates volume memory across parallel geometry units to ensure conflict-free access.

### [Hardware-based Direct Volume Ray Casting](5442-hardware-based-direct-volume-ray-casting.md)
This method implements direct volume ray casting using dedicated programmable Digital Signal Processors (DSPs) to handle the computationally intensive tasks of shading and compositing, decoupled from the geometry generation stage. It solves the bottleneck of real-time volume visualization by offloading the per-voxel light-material interaction calculations from a general-purpose CPU or generic graphics hardware to specialized hardware units. Unlike [Hardware-based Geometry Unit Resampling](3139-hardware-based-geometry-unit-resampling.md), which focuses on data transformation, this method focuses on the optical model execution, specifically supporting the Heidelberg Raytracing algorithm.

### [Custom Hardware Rendering](5746-custom-hardware-rendering.md)
Custom Hardware Rendering refers to direct volume rendering techniques implemented via dedicated, application-specific integrated circuits (ASICs) or field-programmable gate arrays (FPGAs) rather than general-purpose GPUs or CPUs. It solves the problem of achieving real-time interactive frame rates for large, regular volumetric datasets at a time when general-purpose graphics hardware lacked sufficient throughput. Unlike GPU-based methods which rely on programmable fragment shaders, custom hardware units utilize fixed-function pipelines optimized for the highly regular computation patterns of volume rendering, such as shear-warp or ray-marching.

## Background
Custom hardware for volume rendering emerged in the 1990s to address the limitations of CPU-based rendering and early GPUs. The Cube architecture and its successor, VolumePro, are prominent examples of this approach, utilizing hybrid-order algorithms like Shear-Warp [S1, unofficial]. These systems were designed with parallel processing pipelines tailored to specific rendering algorithms, often separating voxel processing (shear) from sample processing (warp and compositing) [S1, unofficial]. Early custom hardware struggled with perspective rendering, often relying on orthographic projections or approximations [S1, unofficial]. As general-purpose GPU throughput increased, the niche for dedicated ASIC/FPGA volume rendering hardware diminished, though the principles remain relevant for specialized embedded or high-performance computing applications [G].

## Sources
[S1] VIRIM Architecture Documentation, URL, 2023-10-01
[S2] Custom Hardware Rendering Overview, URL, 2023-10-01
