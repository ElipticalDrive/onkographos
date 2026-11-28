# Hierarchical Digital Differential Analyzer (HDDA)

Also called: Hierarchical DDA, Multi-resolution Leapfrogging

## Overview
The Hierarchical Digital Differential Analyzer (HDDA) is a ray traversal algorithm designed for sparse, hierarchical volume data structures like OpenVDB [S1, unofficial][S5]. It performs multi-resolution leapfrogging, stepping through nodes and voxels of varying resolutions along a ray to efficiently skip empty space [S1, unofficial][S5]. Unlike standard DDA which uses a fixed step size, HDDA adapts its step size to the current tree level, leveraging the B+tree structure to jump over large empty regions [S3, unofficial][S4]. It is distinguished from its sibling [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md) by its specific implementation of DDA logic at each level of the VDB hierarchy, often optimized via template meta-programming for compile-time fixed tree configurations [S1, unofficial][S5].

## How it works
The HDDA algorithm traverses a VDB grid by maintaining a state that tracks the ray's position in index space and the time parameter $t$ along the ray. It operates on a specified dimension `dim` (a power of two), where `dim=1` corresponds to a voxel and `dim>1` to a tree node [S3, S7, unofficial].

### Inputs and Outputs
*   **Input**: A ray defined by origin $O$, direction $D$, inverse direction $I = 1/D$, and time bounds $[t_{min}, t_{max}]$. A VDB grid accessor providing access to nodes/voxels.
*   **Output**: A sequence of intersected nodes or voxels in front-to-back order, or a boolean indicating if the ray is still within bounds [S6, S3, unofficial].

### Data Structures
The core state is held in a class `HDDA<RayT, CoordT>` [S3, unofficial]. Key fields include:
*   `mDim`: Integer dimension size of the current level (e.g., 1, 8, 16, 32) [S3, S7, unofficial].
*   `mT0`: Current ray parameter (time) [S3, S7, unofficial].
*   `mT1`: Maximum ray parameter [S3, S7, unofficial].
*   `mVoxel`: Current voxel/node index coordinates (aligned to `mDim`) [S3, S7, unofficial].
*   `mStep`: Step direction per axis (+1 or -1) [S3, S7, unofficial].
*   `mDelta`: Time delta per step per axis [S3, S7, unofficial].
*   `mNext`: Time of the next boundary crossing per axis [S3, S7, unofficial].

For GPU implementations (e.g., NanoVDB), additional structures include:
*   `NodeDDA`: Contains `state` (underlying DDA), `origin` (node origin in index space), `dim` (children per axis), `child_size` (child coverage in voxels), and `log2cs` [S6, unofficial].
*   `VdbLeaf`: Contains `color`, `dist` (distance to nearest non-empty node), `num_parents` (hierarchy level 0-3), and `parents` array [S2, unofficial].

### Algorithm Steps

#### 1. Initialization (`init`)
Initialize the HDDA for a specific dimension `dim` [S3, S7, unofficial]:

1.  Set `mDim = dim`, `mT0 = startTime`, `mT1 = maxTime` [S3, S7, unofficial].
2.  Get current position `pos = ray(mT0)` and inverse direction `inv = ray.invDir()` [S3, S7, unofficial].
3.  Compute aligned voxel index: `mVoxel = RoundDown(pos) & (~(dim - 1))` [S3, S7, unofficial].
4.  For each axis `axis` in 0..2:
    *   If `dir[axis] == 0`:
        *   Set `mNext[axis] = Maximum<RealT>::value()` (disabled) [S3, S7, unofficial].
        *   Set `mStep[axis] = 0` [S3, S7, unofficial].
    *   Else if `inv[axis] > 0`:
        *   Set `mStep[axis] = 1` [S3, S7, unofficial].
        *   Set `mNext[axis] = mT0 + (mVoxel[axis] + dim - pos[axis]) * inv[axis]` [S3, S7, unofficial].
        *   Set `mDelta[axis] = inv[axis]` [S3, S7, unofficial].
    *   Else (`inv[axis] < 0`):
        *   Set `mStep[axis] = -1` [S3, S7, unofficial].
        *   Set `mNext[axis] = mT0 + (mVoxel[axis] - pos[axis]) * inv[axis]` [S3, S7, unofficial].
        *   Set `mDelta[axis] = -inv[axis]` [S3, S7, unofficial].

#### 2. Stepping (`step`)
Advance the ray to the next intersected node/voxel boundary [S3, S7, unofficial]:

1.  Find `axis = MinIndex(mNext)` (the axis with the smallest `mNext` value) [S3, S7, unofficial].
2.  **Round-off Correction** (if `ENFORCE_FORWARD_STEPPING` is defined):
    *   If `mNext[axis] <= mT0`:
        *   `mNext[axis] += mT0 - 0.999999f * mNext[axis] + 1.0e-6f` [S3, S7, unofficial].
3.  Update time: `mT0 = mNext[axis]` [S3, S7, unofficial].
4.  Update next hit time: `mNext[axis] += mDim * mDelta[axis]` [S3, S7, unofficial].
5.  Update voxel index: `mVoxel[axis] += mDim * mStep[axis]` [S3, S7, unofficial].
6.  Return `mT0 <= mT1` [S3, S7, unofficial].

#### 3. Level Update (`update`)
Switch to a finer or coarser dimension `dim` [S3, S7, unofficial]:

1.  If `mDim == dim`, return `false` (no change needed) [S3, S7, unofficial].
2.  Compute valid voxel range for the transition:
    *   `voxelMax = (mVoxel + Coord(mDim - 1)) & (~(dim - 1))` [S3, S7, unofficial].
    *   `voxelMin = mVoxel & (~(dim - 1))` [S3, S7, unofficial].
3.  Set `mDim = dim` [S3, S7, unofficial].
4.  Get `pos = ray(mT0)` and `inv = ray.invDir()` [S3, S7, unofficial].
5.  Compute new aligned voxel: `mVoxel = RoundDown(pos) & (~(dim - 1))` [S3, S7, unofficial].
6.  Clamp `mVoxel` to `[voxelMin, voxelMax]` [S3, S7, unofficial].
7.  For each axis `axis` in 0..2:
    *   If `mStep[axis] == 0`, continue [S3, S7, unofficial].
    *   Set `mNext[axis] = mT0 + (mVoxel[axis] - pos[axis]) * inv[axis]` [S3, S7, unofficial].
    *   If `mStep[axis] > 0`, `mNext[axis] += dim * inv[axis]` [S3, S7, unofficial].
8.  Return `true` [S3, S7, unofficial].

#### 4. Hierarchical Traversal Logic
The HDDA is typically embedded in a recursive or iterative traversal of the VDB tree [S6, unofficial]:

1.  Start at the root level (e.g., `InternalNode2` with `dim=32`) [S6, unofficial].
2.  Initialize `NodeDDA` with `child_size` (e.g., 128) [S6, unofficial].
3.  Loop:
    *   Query `NodeDDA` to get local coordinates and child index `idx` [S6, unofficial].
    *   Check if the child is active (e.g., `child_mask[idx]` is set) [S6, unofficial].
    *   If active and inside bounds, recursively traverse the child node or leaf [S6, unofficial].
    *   Step the `NodeDDA` to the next child [S6, unofficial].
4.  At the leaf level, sample the voxel data [S6, unofficial].

### Parameters
*   `dim`: Integer dimension size for the current tree level. `Log2Dim = 0` corresponds to a voxel, `Log2Dim > 0` to a node of size $2^{Log2Dim}$ [S3, S7, unofficial].
*   `ENFORCE_FORWARD_STEPPING`: Macro enabling round-off correction [S3, unofficial].
*   `HDDA_MAX_RAY_STEPS`: Maximum steps (e.g., 1000) [S2, unofficial].
*   `scale`: Array `[1., 8., 128., 4096.]` mapping `num_parents` to step sizes [S2, unofficial].

### Edge Cases and Pitfalls
*   **Zero Direction**: If `dir[axis] == 0`, the axis is disabled to prevent division by zero [S7, unofficial].
*   **Round-off Errors**: Floating-point inaccuracies can cause the ray to step backward or stall. The `ENFORCE_FORWARD_STEPPING` correction prevents this [S3, S7, unofficial].
*   **Non-linear Transforms**: Rays in grids with non-linear transforms (e.g., frustum warping) are not straight in index space, breaking standard HDDA. Solution: Use an auxiliary linear VDB as a mask [S1, unofficial][S5].
*   **Self-Intersection**: A small offset (`4e-4 * step * mask`) is added to the position after each step to prevent hitting the same voxel [S2, unofficial].
*   **Out of Bounds**: Check if `abs(p) > 4096` to handle rays escaping the volume [S2, unofficial].

## Subtopics

### [OpenVDB DDA](2149-openvdb-dda.md)
OpenVDB DDA is a Digital Differential Analyzer specialized for OpenVDB grids, conceptually similar to Bresenham's line algorithm applied to a 3D ray intersecting OpenVDB nodes or voxels. It iterates through grid elements intersected by a ray using a fixed step-size defined by the template parameter `Log2Dim`. Unlike Hierarchical DDA (HDDA) which uses variable step-sizes to skip empty space, OpenVDB DDA marches through every node or voxel boundary along the ray path.

### NanoVolumeHDDA
NanoVolumeHDDA is an iterator that traverses a `NanoGrid` along a ray to yield merged active spans $(t_0, t_1)$ without heap allocations, using a callback interface [S1, unofficial]. It merges adjacent active leaves and tiles to eliminate seam artifacts at 8-voxel tile edges [S1, unofficial]. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by operating specifically on the `NanoGrid` structure and optimizing for zero-allocation iteration [S1, unofficial].

### [RayMajorantIterator](5199-raymajorantiterator.md)
The `RayMajorantIterator` is an iterator that traverses a sparse volume grid using a Hybrid Digital Differential Analyzer (HDDA) to step through voxels or leaves, computing the maximum density (majorant) for each segment where the majorant remains constant. It solves the problem of efficiently determining integration segments for volume path tracing by avoiding per-voxel sampling in empty or uniform regions. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by bundling consecutive steps with identical majorants into single segments, reducing overhead compared to standard voxel DDA which cannot skip inactive nodes or handle variable leaf sizes efficiently.

### [HDDA Hybrid](5201-hdda-hybrid.md)
HDDA Hybrid is a ray traversal algorithm for sparse voxel grids that combines hierarchical skipping with fine-grained traversal. It uses Hierarchical Digital Differential Analyzer (HDDA) to skip empty tiles at resolutions coarser than the leaf level, and switches to standard Digital Differential Analyzer (DDA) to traverse occupied tiles voxel-by-voxel. This approach avoids per-voxel checks in empty space while maintaining sampling accuracy in dense regions. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md).

### HDDA Branch
A ray traversal algorithm using HDDA that samples every tile/voxel, branching based on occupancy [S9, unofficial]. It initializes ray bounds and HDDA, then loops: if the current voxel is active, it records samples and updates time; if not active, it skips time. It steps the HDDA and updates the dimension dynamically [S9, unofficial].

### Teardown Hierarchical Traversal
A hierarchical traversal method that uses DDA within a level and switches levels by rescaling the grid so the new level becomes a level 0 grid, restarting the DDA algorithm [S10, unofficial]. It calculates ray position using $pos = \lfloor (O + t \cdot D) / 2^{level} \rfloor$. Pitfalls include floating-point issues causing outliers in depth/normal buffers and rays traversing corners causing normal problems [S10, unofficial].

### Distance LOD
Reduces traversal granularity for distant objects to save performance while maintaining surface color [S11, unofficial]. Beyond 400 voxels, the DDA terminates at brick granularity, and shading uses the brick's topmost solid material [S11, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of active nodes/voxels intersected by the ray. The hierarchical structure allows skipping large empty regions, making it significantly faster than standard DDA for sparse volumes [S1, unofficial][S5].
*   **Memory Complexity**: $O(1)$ per ray for the traversal state, plus the memory required for the VDB grid structure [S1, unofficial][S5].
*   **Performance**: Tested on RTX 3080 rendering WDAS cloud dataset at 1920x1080 in 6 ms [S8, unofficial].
*   **Strengths**: Efficient empty-space skipping, multi-resolution leapfrogging, low overhead due to template meta-programming [S1, unofficial][S5].
*   **Limitations**: Requires linear transforms for direct use; non-linear transforms require auxiliary grids [S1, unofficial][S5]. Precision issues require careful handling of round-off errors [S3, S7, unofficial].
*   **When to Use**: Sparse volume rendering, level-set intersection, grids with linear transformations [S1, unofficial][S5][S8, unofficial].
*   **When Not to Use**: Dense volumes where the overhead of hierarchy traversal outweighs the benefits, or grids with complex non-linear transforms without auxiliary masks [S1, unofficial][S5].

## Background
The Hierarchical Digital Differential Analyzer was introduced by Ken Museth in the context of the VDB (Volumetric Data Base) data structure [S1, unofficial][S5]. It was developed to efficiently traverse the B+tree structure of VDB grids, taking advantage of compile-time fixed tree configurations [S1, unofficial][S5]. The algorithm is used in OpenVDB and NanoVDB for volume rendering, level-set intersection, and other graphics applications [S1, S3, unofficial][S4]. It has been implemented in various projects, including woxel, Lyr.jl, and Unity HDRP [S2, S6, S8, unofficial].

## Sources
[S1] Hierarchical Digital Differential Analyzer, https://museth.org/Ken/Publications_files/Museth_SIG14.pdf, fetched 2026-10-24 (unofficial)
[S2] NemoInfo/woxel, https://github.com/NemoInfo/woxel, fetched 2026-10-24 (unofficial)
[S3] openvdb/nanovdb/nanovdb/math/HDDA.h at master, https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/math/HDDA.h, fetched 2026-10-25 (unofficial)
[S4] nanovdb/util/HDDA.h File Reference - OpenVDB, https://academysoftwarefoundation.github.io/openvdb/HDDA_8h.html, fetched 2026-10-25
[S5] Hierarchical digital differential analyzer for efficient ray-marching ..., https://www.semanticscholar.org/paper/7dbb75bb242819dc2ddae1a32c38849fb26742ad, fetched 2026-10-26
[S6] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S7] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S8] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S9] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S10] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S11] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
