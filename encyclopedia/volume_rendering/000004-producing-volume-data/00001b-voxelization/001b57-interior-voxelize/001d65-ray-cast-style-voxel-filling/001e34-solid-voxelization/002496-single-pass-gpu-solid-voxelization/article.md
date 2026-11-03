# Single-pass GPU Solid Voxelization

Also called: DX10 voxelization, DX9 voxelization

## Overview
A GPU-based algorithm to voxelize solid geometry in a single pass, producing point-sampled or over/under-conservative results. It differs from boundary-only voxelization by producing a fully filled interior volume and from multi-pass methods by extracting up to 1024 layers in a single geometry pass. Unlike [8, 10], it is accurate even if an arbitrary number of fragments fall in a single voxel and does not require front/back face separation.

## How it works
**Inputs:** A watertight mesh (manifold or topologically coherent mesh not required).
**Outputs:** A 3D voxel grid representing the solid interior.

**Data Structures:**
*   **Slicemap:** A 2D texture storing fragment data. In DX10, this supports multiple render targets (MRT). In DX9, it uses color components.
*   **Voxel Column:** A 32-bit integer (or bitset) representing occupancy for a column of voxels.

**Steps:**
1.  **Rasterization:** Render the input mesh to the slicemap. Each fragment contributes to the voxel column corresponding to its screen-space position.
2.  **Bitmask Accumulation:** Accumulate fragment bitmasks using bitwise XOR. This exploits the Jordan curve theorem: a point is interior if a ray from it intersects the surface an odd number of times.
3.  **Symmetric Difference:** For intersecting watertight models, the algorithm computes the symmetric difference (hollow representation), not the union. This is consistent with the definition of watertight surfaces.
4.  **Density Computation:** Compute density using a kernel of size $2^3$. This is fast and memory efficient because all information is tightly packed.

**Parameters:**
*   **Maximum layers per pass without MRT on DX9:** 32.
*   **Kernel size for density:** $2^3$ [S1].

**Edge Cases and Pitfalls:**
*   **DX9 without MRT:** Performance converges to ~23x speedup over [5] rather than 4x, possibly due to G6 MRT sweet spot.
*   **DX9 Implementation:** Treats all four color components equally, reflecting vector capacities of older processors. It would not scale for DX10 hardware because a lookup texture for a 32-bit integer would need $2^{32}$ entries and not fit into GPU memory. Breaking integers into 16-bit parts would imply two lookups per channel, leading to 64 lookups per 1024-bit voxel column, making bitwise arithmetic more efficient [S1].

## Subtopics

### [Single-pass Solid Voxelization via XOR Bitmask Accumulation](3325-single-pass-solid-voxelization-via-xor-bitmask-accumulation.md)
This method voxelize watertight models in a single GPU render pass by exploiting the Jordan curve theorem: a point is interior if a ray from it intersects the surface an odd number of times. It accumulates fragment bitmasks using bitwise XOR to determine voxel occupancy, distinguishing itself from boundary-only voxelization by producing a fully filled interior volume. It builds on [Single-pass GPU Solid Voxelization](3323-single-pass-gpu-solid-voxelization.md).

## Operating characteristics
**Time and Memory Complexity:**
*   **DX10 (G80) density computation timings:** 0.20 ms at $64^3$, 0.6 ms at $128^3$, 2.9 ms at $256^3$, 20.9 ms at $512^3$ [S1].
*   **DX9 density computation timings:** 0.28 ms at $256^3$, 0.9 ms at $512^3$, 6.5 ms at $1024^3$ [S1].
*   **DX10 speedup over [5]:** Up to >300x for complex models; ~20x for simple cube (12 triangles) [S1].
*   **DX9 speedup over [5]:** ~2x without MRT, converges to 23x for models up to 12,500 triangles [S1].

**Strengths and Limitations:**
*   **Strengths:** High speed for complex models due to multi-layer extraction per pass. Accurate even if arbitrary fragments fall in a single voxel. Does not require front/back face separation.
*   **Limitations:** Fill-rate cost for fragment output and blending. Simple models (low geometric complexity, high fill-rate) are worst-case scenarios because standard rendering is very cheap.

**When to use:**
*   When high-speed voxelization of complex watertight models is required.
*   When MRT support is available (DX10) or when older cards without MRT are used (DX9).

**When not to use:**
*   When union of intersecting models is required (algorithm computes symmetric difference).
*   When manifold or topologically coherent mesh is not available (watertight input suffices, but non-watertight may produce unexpected results).

## Background
Originated in the paper "Single-pass GPU Solid Voxelization and Applications" [S1]. The approach is simple but efficient single-pass solutions were previously non-existent. It is appealing due to its accuracy and simplicity.

## Sources
[S1] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26
