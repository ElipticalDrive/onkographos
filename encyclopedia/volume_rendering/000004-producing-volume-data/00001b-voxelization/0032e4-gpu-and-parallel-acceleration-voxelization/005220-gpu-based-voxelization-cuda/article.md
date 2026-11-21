# GPU-based Voxelization (CUDA)


## Overview
GPU-based Voxelization accelerates the conversion of polygon meshes or geometric primitives into discrete volumetric representations by leveraging the massive parallelism of CUDA-capable GPUs [G]. It solves the performance bottleneck of CPU-based rasterization, enabling real-time voxelization of high-polygon models [G]. It distinguishes itself from sibling methods by its direct use of CUDA kernels for parallel triangle processing and atomic memory updates, contrasting with CPU parallelization [7135-parallel-cpu-voxelization.md] or specialized structures like linked lists [1187-per-pixel-linked-list-voxelization.md].

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Vessel Voxelizer](2321-vessel-voxelizer.md)
Converts vascular line segments into fuzzy voxel volumes using supersampling to estimate volume fractions. It distinguishes itself from standard binary voxelization by estimating continuous volume intersections, making it suitable for rendering translucent vascular networks.

### [Flexible Dual Grid Mesh Voxelization](2640-flexible-dual-grid-mesh-voxelization.md)
Determines active sparse voxels and solves an enhanced Quadratic Error Function (QEF) for dual vertex positions. It serves as the foundational occupancy and geometry reconstruction stage for O-Voxel, differing from siblings by strictly providing raw voxel indices and dual vertices without higher-level serialization.

### [CUDA Voxelizer GPU Voxelization](6757-cuda-voxelizer-gpu-voxelization.md)
An optimized implementation of Schwarz and Seidel's algorithm for surface and solid voxelization. It distinguishes itself by using bit-packed storage formats and conservative rasterization techniques adapted for GPU atomics, enabling real-time performance for interactive applications.

### [GPU-based Voxel Query via Conditional Histogram Computation](10393-gpu-based-voxel-query-via-conditional-histogram-computation.md)
Uses conditional histograms for efficient voxel queries. It distinguishes itself by focusing on query operations rather than direct mesh-to-voxel conversion.

## Background
The method is based on the algorithm described by Schwarz and Seidel [6757-cuda-voxelizer-gpu-voxelization.md]. It was implemented as a high-performance, real-time solution leveraging CUDA parallelism [6757-cuda-voxelizer-gpu-voxelization.md]. Atomic operations are a key component, though they can introduce contention on older hardware architectures [G].

## Sources
