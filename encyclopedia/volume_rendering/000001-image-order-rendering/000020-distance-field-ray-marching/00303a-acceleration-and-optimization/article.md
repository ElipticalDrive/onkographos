# Acceleration and Optimization


## Overview
This family covers techniques to reduce the computational cost of ray marching and volume traversal, addressing the bottleneck of excessive scene queries or ray steps [G]. It sets itself apart from sibling methods like [Beam Optimization](932-beam-optimization.md) and [Performance Optimizations for SDF Ray Marching](1629-performance-optimizations-for-sdf-ray-marching.md) by focusing on reducing the number of march steps or the cost of each step through geometric culling, resolution scaling, and parameter tuning, rather than changing the fundamental traversal structure or parallelism model [G].

## Shared principles
Builds on [Distance-field ray marching](21-distance-field-ray-marching.md). The primary cost driver is the number of iterations required to reach a surface or exit the scene [G]. Optimization strategies generally aim to:
1. Skip scene evaluations when the ray is known to be far from geometry (culling/bounding) [G].
2. Reduce the total number of pixels processed (resolution scaling) [G].
3. Increase step size in empty space (over-relaxation), accepting potential precision loss [G].
4. Group rays to exploit spatial coherence (beam optimization) [G].

## Subtopics

### [Beam Optimization](932-beam-optimization.md)
An acceleration technique for primary rays that groups adjacent rays into beams to reduce redundant intersection tests. By marching a group of rays together along a central path, the method exploits spatial coherence: if the distance to the nearest geometry is large relative to the group's spatial extent, all rays in the group can advance simultaneously without individual checks. This reduces the number of scene queries for rays far from geometry, at the cost of potential over-stepping or splitting when rays diverge near surfaces. Choose this when rendering large images with high spatial coherence in primary rays.

### [Performance Optimizations for SDF Ray Marching](1629-performance-optimizations-for-sdf-ray-marching.md)
A set of techniques designed to reduce the computational cost of rendering scenes defined by SDFs while maintaining visual fidelity. It solves the primary bottleneck of ray marching: the high number of SDF evaluations and shader instructions required per pixel, particularly in complex or high-resolution scenes. What sets these optimizations apart from sibling methods like [Beam Optimization](932-beam-optimization.md) is their focus on reducing the number of march steps or the cost of each step through geometric culling, resolution scaling, and parameter tuning, rather than changing the fundamental traversal structure or parallelism model. Choose this for general real-time SDF rendering where visual fidelity must be balanced with performance.

### [Bounding Shape Optimization](7750-bounding-shape-optimization.md)
Optimizes volumetric rendering by skipping density evaluations in empty space using pre-defined bounding shapes. Choose this when scenes contain large empty regions or simple bounding volumes that can quickly reject rays.

### Block Culling
#### definition
An algorithm that culls blocks to reduce the time to reach content values inside RGBA volumes, particularly effective when sparsity results from runtime parameters like narrower allowed density ranges [S1, unofficial].
#### pitfall
Block culling does not provide significant performance regression when there are no blocks it can impact, specifically when the user-requested content range is greater than or equal to the actual content range [S1, unofficial].
#### number
In block traversal mode, 'RT+Cull' achieved 4.823ms on Fish (c) vs 7.375ms for 'RT' [S1, unofficial]. In blend mode, 'RT+Cull' achieved 8.079ms on Fish (c) vs 10.798ms for 'RT' [S1, unofficial].

## Background
Block culling and related traversal optimizations are discussed in the context of sparse volume rendering using hardware ray tracing and block-based data structures [S1, unofficial].

## Sources
[S1] [PDF] Sparse Volume Rendering using Hardware Ray Tracing and Block ..., https://gpuopen.com/download/SA2021_BlockWalk.pdf, fetched 2026-10-26 (unofficial)
