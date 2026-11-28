# Hardware-Accelerated and GPU Implementations

Also called: GPU Volume Ray Casting, Hardware-Accelerated DVR

## Overview
This family encompasses methods that offload the computationally intensive sampling and compositing stages of direct volume ray casting to the GPU, utilizing programmable shaders and texture hardware to achieve interactive frame rates [S1]. It solves the performance bottleneck of CPU-based ray casting by exploiting the intrinsic parallelism of fragment processors [G]. It differs from [Shear-Warp](shear-warp.md) by avoiding the final warp step and performing classification after resampling, and from [Custom Hardware Units](8284-custom-hardware-units.md) by relying on general-purpose programmable GPUs rather than dedicated ASICs [S1].

## Shared principles
All methods in this family implement the emission-absorption optical model defined in [Direct volume ray casting](19-direct-volume-ray-casting.md) [G]. The core pipeline for each pixel consists of ray generation, hardware-accelerated sampling, classification, and compositing [G].

1.  **Ray Generation**: Rays are generated per-pixel, typically by computing ray parameters from camera matrices in vertex/fragment shaders or by rendering the volume's bounding box geometry to determine entry/exit points [G].
2.  **Hardware-Accelerated Sampling**: Scalar volume values are sampled using the GPU's texture hardware. The volume data is stored in 3D texture memory, and the texture engine performs trilinear interpolation of the eight surrounding voxels in parallel, avoiding manual shader interpolation [S1].
3.  **Fragment Shader Compositing**: The ray marching loop, including classification and front-to-back compositing, is executed within the fragment shader [G]. The standard compositing equation updates accumulated color $C'$ and opacity $\alpha'$:
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
4.  **Early Ray Termination (ERT)**: Rays are terminated when accumulated opacity exceeds a threshold (typically 0.95 or 1.0) to optimize performance [G].

## Subtopics

[Direct Volume Rendering](1184-direct-volume-rendering.md): Direct Volume Rendering (DVR) is an image-order technique that renders 3D volumetric data directly without converting it to surface geometry. It treats the volume as a translucent gel or gas, simulating light absorption and emission as rays pass through the scalar field. Unlike indirect methods that extract intermediate geometric primitives like meshes, DVR processes the entire dataset voxel-by-voxel, preserving all internal information and allowing weak or ill-defined structures to remain visible. It is distinguished from its sibling [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-volume-rendering.md) by its specific reliance on GPU-based ray casting within the fragment shader, leveraging hardware texture interpolation for sampling [G].

[GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md): The GPU Multi-Volume Ray Casting Mapper is a volume rendering technique that renders multiple volume datasets simultaneously by extending the single-volume mapper to accept multiple datasets and volume property objects, using GPU shaders for ray traversal and compositing. It sets itself apart from its siblings by compositing multiple 3D scalar fields along a single ray, using coordinate transformation matrices to align samples from different volumes to a common ray space. It builds on [Hardware-Accelerated and GPU Implementations](9525-hardware-accelerated-and-gpu-implementations.md) by providing the specific logic for multi-input data management and shader uniform updates required for simultaneous rendering [G].

[Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-volume-rendering.md): Hardware-Accelerated Volume Rendering is a direct volume ray casting technique that offloads the computationally intensive sampling and compositing steps to the GPU's texture hardware and fragment shaders. It solves the performance bottleneck of software-based ray casting by utilizing 3D texture memory for trilinear interpolation and programmable shaders for front-to-back compositing. It differs from [Shear-Warp](shear-warp.md) by avoiding the final warp step and performing classification after resampling, and from [Custom Hardware Units](8284-custom-hardware-units.md) by relying on general-purpose programmable GPUs rather than dedicated ASICs [G].

[WebGL Multi-pass Ray-casting with 3D Geometry Intersection](6575-webgl-multi-pass-ray-casting-with-3d-geometry-intersection.md): This method enables interactive volume rendering of 3D volumetric datasets coexisting with 3D triangular meshes within the constraints of the WebGL API. It solves the problem of rendering hybrid scenes (volumes + geometry) on web platforms that lack native 3D texture support and often lack robust depth texture support. Unlike standard ray-casting which only samples volume data, this method integrates 3D surface occlusion and color by using a multi-pass pipeline to pre-compute depth and color information for both the volume bounding box and the 3D meshes, allowing the final ray-casting pass to correctly handle intersections and alpha blending. It is distinguished from single-pass WebGL ray-casting by its explicit handling of 3D geometry occlusion through depth buffering [G].

[Volume Ray-Casting on Graphics Hardware](7443-volume-ray-casting-on-graphics-hardware.md): This method implements direct volume ray casting on programmable graphics hardware using a multi-pass stream model. It solves the performance bottleneck of CPU-based volume rendering by exploiting the intrinsic parallelism of fragment processors and efficient inter-pass communication via texture render targets. Unlike single-pass rasterization, this approach iteratively accumulates optical properties across multiple rendering passes, enabling effective early ray termination and empty-space skipping through hardware-supported early z-tests [G].

[GPU-based Tiled Ray Casting with Depth Peeling](9631-gpu-based-tiled-ray-casting-with-depth-peeling.md): This method is a hardware-accelerated ray casting algorithm for volume rendering of time-varying scalar fields on unstructured tetrahedral meshes. It solves the problem of rendering non-convex meshes on GPUs by combining screen tiling for performance with depth peeling to correctly handle ray re-entry into the mesh. It sets itself apart from its sibling [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-and-gpu-implementations.md) by explicitly addressing non-convexity without convexification, and from [WebGL Path Tracing Volume Rendering](8592-webgl-path-tracing-volume-rendering.md) by using deterministic ray casting on unstructured grids rather than path tracing on regular grids [G].

[Hardware-assisted Volume Deformation](9864-hardware-assisted-volume-deformation.md): Hardware-assisted volume deformation renders non-linearly deformed volumes by integrating the deformation process directly into the fragment shader, avoiding the computational expense of pre-computing a resampled deformed volume. Unlike methods that pre-compute the deformed volume, this approach allows for dynamic deformations at interactive rates by evaluating the inverse deformation function per-pixel. It builds on [Hardware-Accelerated and GPU Implementations](9525-hardware-accelerated-and-gpu-implementations.md) by modifying the texture coordinate generation and sampling logic to account for spatial transformations [G].

[Hardware-Accelerated AMR Volume Rendering via Homogenization](10503-hardware-accelerated-amr-volume-rendering-via-homogenization.md): This method renders Adaptive Mesh Refinement (AMR) data by preprocessing the hierarchy into a set of non-overlapping Cartesian grid patches, allowing standard Direct Volume Ray Casting (DVR) algorithms to be used without resampling. It differs from methods that resample AMR data to a uniform grid or render individual grids with complex boundary handling by explicitly tiling the domain with homogeneous resolution regions [G].

[GPU Based Single-Pass Raycasting](10854-gpu-based-single-pass-raycasting.md): GPU Based Single-Pass Raycasting is a volume rendering technique where the volume is traversed along a single ray per pixel, implemented within the Volume Rendering Component framework. It solves the limitations of polygonal-based methods (which are limited in expressive power) and slicing techniques (which can lead to artifacts) by providing high-quality visual results. It is distinguished from its siblings by its specific implementation as a single-pass fragment shader execution on modern graphics hardware, relying on hardware-accelerated 3D texture sampling for trilinear interpolation [G].

[GPU Implementation Architecture](11245-gpu-implementation-architecture.md): This article describes the standard software architecture for implementing direct volume ray casting on modern GPUs using OpenGL and GLSL. It details the pipeline for generating spherical harmonic (SH) coefficient volumes and performing ray marching via fragment shaders. It distinguishes itself from generic hardware-accelerated approaches by focusing on the specific data structures (3D coefficient volumes) and rendering techniques (slice-by-slice ray marching with Monte Carlo sampling) required for advanced light-material interaction [G].

### Brick Boundary Clamping
#### definition
- A technique to clamp the last ray sample position to the volume or brick extents to prevent artifacts like holes in isosurfaces caused by tri-linear interpolation at brick boundaries [S1].
#### step
- Clip each ray against the boundaries of the volume/brick using a method similar to Sutherland-Hodgman polygon clipping to find the intersection with the back face [S1].
- Repeat the clipping process for the y- and z-axes to yield the final intersection of the ray and the back face [S1].
#### code
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clamp_last_sample(pos, vol_extent_min, vol_extent_max, offset):
    scale = 0.0
    temp = vol_extent_max - pos
    if temp.x < 0:
        scale_x = temp.x
    temp = pos - vol_extent_min
    if temp.x < 0:
        scale_x = temp.x
    correction = abs(scale_x / offset.x)
    pos.x -= correction * offset.x
    return pos
```
#### pitfall
- Using one voxel overlap between bricks and correct last sampling points is insufficient when using pre-integration because different sampling distances occur. This is overcome by using incremental pre-integration tables as presented by Weiler et al. [WKME03] [S1].

### Multi-GPU Volume Rendering
#### definition
- A strategy to increase model capacity with multiple GPUs by partitioning 3D space into non-overlapping tiles, where each GPU owns a single tile's NeRF. Rendering and training avoid blending by ensuring sample intervals do not extend beyond tile boundaries [S2].
#### step
- Distribute ray samples among GPUs based on the bounding box of the tiles, ensuring sample intervals do not extend beyond tile boundaries [S2].
- Query sample attributes (color and density) on each respective GPU [S2].
- Perform volume rendering through a global gather operation, consolidating information across all GPUs onto a single GPU to compute the final pixel color [S2].
- During training forward pass, each GPU performs a global gather (broadcast) to obtain color and density from all other GPUs [S2].
- Each GPU computes the loss locally and back-propagates gradients to its own parameters. No gradient communication is required [S2].
#### complexity
- Requires synchronizing $O(SK^2)$ data across GPUs, where $K$ is the number of GPUs and $S$ is the average number of samples per-ray per-GPU [S2].

### Plane Composing with Hardware Acceleration
#### step
- Send volume planes one by one as transparent textures to the AGP (Accelerated Graphics Port) using graphic accelerators. Compose back to front [S3, unofficial].

### Fixed-Point Volume Ray Casting
#### definition
- A volume rendering technique implemented by VTK that uses fixed-point arithmetic for ray casting to render volumetric data, specifically medical images [S4, unofficial].
#### parameter
- Not given in the sources.
#### layout
- Not given in the sources.

## Background
The shift from CPU-based to GPU-based volume rendering began in the early 2000s with the advent of programmable graphics pipelines (e.g., DirectX 9, OpenGL 2.0) [G]. Early implementations utilized multi-pass rendering techniques to overcome hardware limitations on dependent texture fetches and floating-point precision [G]. The introduction of 3D texture hardware and fragment shaders allowed for single-pass ray marching, significantly improving performance and enabling interactive visualization [S1]. Recent developments focus on scaling these techniques to large datasets using multi-GPU architectures and handling complex data structures like AMR and unstructured meshes [S1, S2].

## Sources
[S1] Optimized volume raycasting for graphics-hardware-based cluster systems, https://doi.org/10.2312/EGPGV/EGPGV06/059-066, fetched 2026-10-27
[S2] NeRF-XL: Scaling NeRFs with Multiple GPUs, https://doi.org/10.1007/978-3-031-73195-2_6, fetched 2026-10-26
[S3] Introduction to volume rendering, https://slidetodoc.com/introduction-to-volume-rendering-presented-by-zvi-devir/, fetched 2026-10-29 (unofficial)
[S4] comedate/VolumeRendering, https://github.com/comedate/VolumeRendering, fetched 2026-10-27 (unofficial)
