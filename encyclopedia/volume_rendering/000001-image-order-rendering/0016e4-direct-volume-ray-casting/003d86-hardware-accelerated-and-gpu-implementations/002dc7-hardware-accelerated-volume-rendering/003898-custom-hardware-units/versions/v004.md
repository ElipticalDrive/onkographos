# Custom Hardware Units
Also called: Dedicated Volume Rendering Hardware, Fixed-Function Volume Rendering Accelerators

## Overview
Custom Hardware Units implement direct volume ray casting by offloading the entire rendering pipeline to dedicated application-specific integrated circuits (ASICs) or field-programmable gate arrays (FPGAs), rather than relying on general-purpose CPUs or programmable GPUs [S2, unofficial]. This family addresses the historical computational bottleneck where software implementations consumed excessive time on geometry calculations and compositing, enabling real-time frame rates for regular data sets through highly parallel, fixed-function logic [S1, unofficial]. Unlike general-purpose GPU implementations that use texture sampling and fragment shaders, these units employ custom address generation, precalculated weight memories, and dedicated Digital Signal Processors (DSPs) to handle ray casting, shading, and compositing in hardware [S1].

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

### [Hardware-based Geometry Unit Resampling](3139-hardware-based-geometry-unit-resampling.md)
Offloads perspective calculations, gradient estimation, and volume data resampling to dedicated hardware accelerators using custom address generation and precalculated weight memories. It solves the bottleneck where software implementations consume up to 80% of computation time on these tasks. Choose this when high-throughput gradient estimation and backward mapping interpolation are required without general-purpose shader overhead.

### [Hardware-based Direct Volume Ray Casting](5442-hardware-based-direct-volume-ray-casting.md)
Offloads ray casting, shading, and compositing to dedicated programmable DSPs, decoupling these operations from general-purpose processors. It uses specialized DSPs to compute light-material interactions with high throughput, distinct from tree-based hardware methods by using sequential scanline processing per DSP. Choose this for high-throughput light-material interaction calculations in regular grid data.

### [Custom Hardware Rendering](5746-custom-hardware-rendering.md)
Implements the entire volume rendering pipeline via dedicated ASICs or FPGAs, addressing the need for real-time rendering when CPUs/GPUs lacked sufficient throughput. It achieves interactive frame rates through highly parallel, regular computation, often using multiple identical pipelines. Choose this for fixed-function, high-performance rendering of regular datasets where flexibility is secondary to speed.

## Background
Custom hardware units emerged in the 1990s and early 2000s to address the limitations of CPU-based volume rendering, which was too slow for interactive applications [S1, unofficial]. Architectures such as Cube, Vizard, and VolumePro utilized dedicated hardware to achieve real-time performance for regular grid data [S1, unofficial]. These systems were largely superseded by the rise of programmable GPUs, which offered greater flexibility and comparable performance for many tasks, though custom hardware remains relevant in specific high-throughput or low-latency applications [S2, unofficial].

## Sources
[S1] The provided notes for the family and child articles.
[S2] The provided notes for the family and child articles.
