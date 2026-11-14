# GPU-based Voxelization (CUDA)

Also called: CUDA Voxelization, GPU Voxelization

## Overview
GPU-based Voxelization accelerates the conversion of polygon meshes or geometric primitives into discrete volumetric representations by leveraging the massive parallelism of CUDA-capable GPUs [S1, S3]. It solves the performance bottleneck of CPU-based rasterization, enabling real-time voxelization of high-polygon models [S3]. It distinguishes itself from sibling methods by its direct use of CUDA kernels for parallel triangle processing and atomic memory updates, contrasting with CPU parallelization [7135-parallel-cpu-voxelization.md] or specialized structures like linked lists [1187-per-pixel-linked-list-voxelization.md].

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Vessel Voxelizer](2321-vessel-voxelizer.md)
Converts vascular line segments into fuzzy voxel volumes using supersampling to estimate volume fractions [2321-vessel-voxelizer.md]. Suitable for translucent vascular networks; uses fixed $8 \times 8 \times 8$ thread blocks and $K=10$ supersampling [2321-vessel-voxelizer.md].

### [Flexible Dual Grid Mesh Voxelization](2640-flexible-dual-grid-mesh-voxelization.md)
Determines active sparse voxels and solves an enhanced Quadratic Error Function (QEF) for dual vertex positions, supporting non-manifold surfaces [2640-flexible-dual-grid-mesh-voxelization.md]. Serves as the foundation for O-Voxel representations; uses hashmaps for occupancy [2640-flexible-dual-grid-mesh-voxelization.md].

### CUDA Voxelizer
An optimized implementation of Schwarz and Seidel's algorithm for surface and solid voxelization [6757-cuda-voxelizer-gpu-voxelization.md]. Uses bit-packed grids and atomic operations; requires watertight meshes for solid mode [6757-cuda-voxelizer-gpu-voxelization.md].

### GPU-based Voxel Query via Conditional Histogram Computation
Uses conditional histograms for voxel queries [10393-gpu-based-voxel-query-via-conditional-histogram-computation.md].

## Background
The method is based on the algorithm described by Schwarz and Seidel [S1]. It was implemented as a high-performance, real-time solution leveraging CUDA parallelism [S3]. Atomic operations are a key component, though they can introduce contention on older hardware architectures [S1].

## Sources
[S1] Schwarz and Seidel, "Real-Time Software Rasterization and Hardware Aided Accelerations," 2002.
[S2] CUDA Voxelizer Implementation Details, Unofficial.
[S3] CUDA Voxelizer GPU Voxelization, Unofficial.
