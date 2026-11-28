# Depth and Layer-Based Rendering

Also called: Depth-based Volume Rendering, Layered Volume Rendering

## Overview
Depth and Layer-Based Rendering optimizes direct volume ray casting by decoupling the determination of ray entry and exit points from the volumetric sampling process, or by explicitly managing multiple non-overlapping ray segments per pixel. This family solves the performance bottlenecks of standard ray casting, such as redundant sampling in empty space and the inability to handle embedded geometric objects or multiple transfer functions in a single pass [S1, unofficial]. It differs from standard [Direct volume ray casting](19-direct-volume-ray-casting.md) by using depth-buffered geometry rendering or pre-computed layer buffers to define precise integration intervals, enabling early fragment rejection and efficient handling of complex spatial overlaps.

## Shared principles
All methods in this family share the following core mechanism, building on [Direct volume ray casting](19-direct-volume-ray-casting.md):

1.  **Ray Interval Definition**: Instead of sampling the entire bounding box or a fixed number of steps, the method determines specific start ($t_{start}$) and end ($t_{end}$) parameters for the ray segment to be integrated. These intervals are derived either from geometric depth tests against the volume boundary (Two-Pass) or from a pre-computed buffer of non-overlapping segments (Multi-Layer) [S1, unofficial].
2.  **Segmented Integration**: The volume rendering equation is applied only within the defined intervals $[t_{start}, t_{end}]$. Samples are taken at positions $t_i = t_{start} + i \cdot \Delta t$ where $\Delta t = (t_{end} - t_{start}) / N_{seg}$ [S1, unofficial].
3.  **Early Termination**: Because the intervals often correspond to visible or opaque regions, these methods frequently employ early termination when accumulated opacity $\alpha_{acc}$ reaches 1.0, skipping remaining samples in the segment [S1, unofficial].
4.  **Material/Transfer Function Binding**: In multi-layer variants, each segment is associated with a specific `materialID` or VOI priority, allowing the use of distinct transfer functions or shading models for different spatial regions (e.g., embedded objects vs. volume data) [S1, unofficial].

## Subtopics

### [Two-Pass Depth Ray Casting](3377-two-pass-depth-ray-casting.md)
A GPU-based technique that determines ray entry and exit points by rendering the volume's bounding box geometry twice, then accumulates color along those rays in subsequent passes. It differs from standard ray casting by explicitly decoupling geometric intersection calculation from volumetric sampling, using depth-buffered rendering of the bounding box to provide precise start and end coordinates for each pixel's ray. This approach enables early fragment rejection via depth testing when opacity thresholds are met, optimizing performance for opaque regions. See [Two-Pass Depth Ray Casting](3377-two-pass-depth-ray-casting.md).

### [Multi-layer Volume Ray Casting](9441-multi-layer-volume-ray-casting.md)
A unified rendering framework that treats volume datasets and geometric objects (clipping geometry, embedded opaque/semi-transparent objects, editing masks, MPRs, and non-empty-space boundaries) as Volume-of-Interests (VOIs). It solves the problem of handling spatial overlaps, empty-space skipping, and multiple material-specific transfer functions in a single pass by encoding VOI interactions into a pre-generated ray-layer buffer. Unlike standard single-layer ray casting, which cannot skip empty spaces between visible samples or handle embedded objects producing multiple layers, this method explicitly stores non-overlapping ray segments per pixel. It differs from polygon-assisted ray casting and single-pass ray casters that store full 3D texture coordinates by storing only the Z-coordinate of the starting position, reducing memory usage and ensuring sampling alignment across bricks. See [Multi-layer Volume Ray Casting](9441-multi-layer-volume-ray-casting.md).

### [Depth Alpha Blending Projection](19551-depth-alpha-blending-projection.md)
Renders a 2D projection of a 3D volume by alpha blending voxel values along rays, assuming pre-multiplied alpha. It offers two variants: a grayscale "Depth-8" mode that outputs a single channel, and an RGB mode that splits the ray into two halves for accumulation to approximate depth-aware color blending. It is distinguished from its siblings by its explicit two-pass accumulation structure in the RGB variant, which allows for specific depth-weighting strategies not present in standard single-pass ray casting. See [Depth Alpha Blending Projection](19551-depth-alpha-blending-projection.md).

### Layered Depth Image (LDI) extraction
#### definition
A view-dependent method that reduces the traversal number to the scene depth complexity [S1, unofficial].
#### pitfall
It requires downloading depth images from video memory and resorting depth lists pixel-by-pixel on the CPU, affecting efficiency based on volume resolution [S1, unofficial].

## Background
The concept of using depth information to optimize volume rendering has evolved alongside GPU hardware capabilities. Early implementations focused on using depth buffers to determine entry and exit points to avoid sampling empty space . The multi-layer approach was developed to handle complex scenes with embedded geometry and multiple materials, shifting computational workload to layer sorting to keep the ray-casting loop simple [S1, unofficial]. The LDI approach was introduced as a voxelization alternative, though it noted significant CPU-side overhead for depth list resorting [S1, unofficial].

## Sources
[S1] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
