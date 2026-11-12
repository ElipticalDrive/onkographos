# Regular Grid Traversal

Also called: Grid Traversal Algorithm

## Overview
Regular Grid Traversal is a ray-grid traversal method that accelerates ray-primitive intersection tests by mapping primitives to a uniform 3D voxel grid and stepping through occupied cells using 3D-DDA [S1, 3950-regular-grid]. It solves the problem of redundant intersection tests in dense scenes by skipping empty space, reducing the number of primitive intersection checks [3950-regular-grid]. Unlike [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) which typically renders volumetric density, this method accelerates geometric ray tracing by storing primitive pointers in voxels [3950-regular-grid]. It differs from [Hierarchical Grid Traversal](52-name.md) by using a flat, uniform structure without hierarchical levels, and from [Sparse Voxel Traversal](5228-sparse-voxel-tracing.md) by not using sparse data structures like octrees [3950-regular-grid].

## Shared principles
The family shares the mechanism of 3D Digital Differential Analyzer (3D-DDA) traversal to step through a uniform 3D grid [4091-voxel-based-ray-tracing-for-gi-and-specular-reflections, 4240-optimised-voxel-ray-marching]. The grid partitions the scene's Axis-Aligned Bounding Box (AABB) into $N^3$ uniform cells [3950-regular-grid]. Rays are advanced by calculating the distance to the next cell boundary along each axis, advancing the coordinate with the smallest distance [3950-regular-grid, 4091-voxel-based-ray-tracing-for-gi-and-specular-reflections]. The traversal stops when the ray exits the grid bounds or finds a primitive intersection [3950-regular-grid].

## Subtopics

### [Regular Grid](3950-regular-grid.md)
A standard implementation using a flat 1D array of primitive lists. It is optimal for uniform data distribution on GPUs due to cache efficiency [3950-regular-grid]. Choose it for general-purpose ray tracing where scene density is uniform.

### [Voxel-based Ray Tracing for GI and Specular Reflections](4091-voxel-based-ray-tracing-for-gi-and-specular-reflections.md)
Uses a 128³ occupancy grid to sample Light Propagation Volumes (LPV) for indirect lighting and reflection fallbacks [4091-voxel-based-ray-tracing-for-gi-and-specular-reflections]. Choose it for global illumination and screen-space reflection fallbacks.

### [Optimised Voxel Ray Marching](4240-optimised-voxel-ray-marching.md)
Optimizes traversal by caching chunk references and using bit-shifts for indexing, avoiding repeated calculations [4240-optimised-voxel-ray-marching]. Choose it for sparse voxel worlds with fixed chunk sizes to reduce computational overhead.

## Background
The method originates from early ray tracing research, where grid traversal algorithms were developed to determine cells traversed by a ray [S1]. The algorithm is similar to rasterization but must find all traversed cells, unlike Bresenham's line algorithm which produces visually acceptable lines without finding all pixels [S1]. The performance is modeled by two constants: $T_{setup}$ (setup time) and $T_{step}$ (time to advance between cells) [S1]. Low $T_{setup}$ benefits low-resolution grids, while high $T_{step}$ hurts performance in larger grids [S1].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
