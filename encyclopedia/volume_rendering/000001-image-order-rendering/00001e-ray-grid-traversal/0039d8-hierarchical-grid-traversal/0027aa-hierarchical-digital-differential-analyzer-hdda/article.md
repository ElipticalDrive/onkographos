# Hierarchical Digital Differential Analyzer (HDDA)


## Overview
The Hierarchical Digital Differential Analyzer (HDDA) is a ray traversal algorithm that performs multi-resolution leapfrogging through sparse volume data structures, specifically OpenVDB and NanoVDB grids [S1, S3, unofficial]. It adapts the Digital Differential Analyzer (DDA) to step through nodes or voxels of varying power-of-two dimensions, skipping empty space by jumping between grid nodes of different resolutions [S2, S6, unofficial]. Unlike standard voxel DDA which checks every voxel, HDDA leverages the hierarchical tree structure (B+tree) to traverse coarse tiles first and only descend into occupied children, significantly accelerating ray-marching for sparse volumes [S1, unofficial][S4]. It is distinct from [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md) in its specific implementation for VDB structures using compile-time template meta-programming to inline the hierarchy, and from [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md) by focusing on traversal efficiency rather than shading interpolation [S1, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$, direction $\mathbf{d}$, inverse direction $\mathbf{i} = 1/\mathbf{d}$, and time bounds $[t_{min}, t_{max}]$; a hierarchical grid (e.g., VDB/NanoVDB) with fixed tree configuration at compile-time; current tree level dimension $dim$ (where $dim = 2^{Log2Dim}$).
*   **Output:** A sequence of intersected voxel/node indices and their corresponding ray parameters $t$, in front-to-back order [S5, unofficial].

**Data Structures:**
The core state is held in the `HDDA` class [S3, S6, unofficial]:
*   `mDim` (`int32_t`): The dimension size of the current node/voxel being traversed.
*   `mT0` (`RealT`): Current ray parameter (time) at the start of the current step.
*   `mT1` (`RealT`): Maximum ray parameter (end of ray).
*   `mVoxel` (`CoordT`): 3D integer coordinate of the current node/voxel origin.
*   `mStep` (`CoordT`): Step direction per axis ($+1, -1, 0$).
*   `mDelta` (`Vec3T`): Time delta per step per axis ($\Delta t$).
*   `mNext` (`Vec3T`): Time of the next boundary crossing per axis.

For hierarchical traversal, a `NodeDDA` struct is used for internal nodes [S5, unofficial]:
*   `state`: Underlying DDA with `voxel_size = child_size`.
*   `origin`: Node origin in index space.
*   `dim`: Children per axis (e.g., 8, 16, 32).
*   `child_size`: Child coverage in voxels (e.g., 1, 8, 128).
*   `log2cs`: $\log_2(child\_size)$ for bit-shift index computation.

**Algorithm Steps:**

1.  **Initialization (`init`):**
    Given ray $\mathbf{o}, \mathbf{d}, \mathbf{i}$, bounds $t_{min}, t_{max}$, and dimension $dim$:
    1.  Set `mDim = dim`, `mT0 = t_min`, `mT1 = t_max`.
    2.  Compute current position $\mathbf{p} = \mathbf{o} + t_{min} \mathbf{d}$.
    3.  Compute aligned voxel origin: $\mathbf{mVoxel} = \lfloor \mathbf{p} \rfloor \ \& \ \sim(dim - 1)$ [S3, S6, unofficial].
    4.  For each axis $a \in \{0, 1, 2\}$:
        *   If $d[a] == 0$:
            *   `mStep[a] = 0`
            *   `mNext[a] = +\infty` (disabled)
        *   Else if $i[a] > 0$ (ray moving in positive direction):
            *   `mStep[a] = 1`
            *   `mNext[a] = t_{min} + (mVoxel[a] + dim - p[a]) \times i[a]`
            *   `mDelta[a] = i[a]`
        *   Else ($i[a] < 0$, ray moving in negative direction):
            *   `mStep[a] = -1`
            *   `mNext[a] = t_{min} + (mVoxel[a] - p[a]) \times i[a]`
            *   `mDelta[a] = -i[a]`

2.  **Step (`step`):**
    1.  Find axis $a = \text{argmin}_k(mNext[k])$ [S3, S6, unofficial].
    2.  **Round-off Correction:** If `ENFORCE_FORWARD_STEPPING` is defined and `mNext[a] <= mT0`:
        *   `mNext[a] += mT0 - 0.999999f * mNext[a] + 1.0e-6f` [S3, S6, unofficial].
    3.  Update time: `mT0 = mNext[a]`.
    4.  Update next hit time for axis $a$: `mNext[a] += mDim * mDelta[a]`.
    5.  Update voxel index: `mVoxel[a] += mDim * mStep[a]`.
    6.  Return `mT0 <= mT1`.

3.  **Update (`update`):**
    Used when descending to a child level with dimension $dim_{child}$ [S3, S6, unofficial]:
    1.  If `mDim == dim_{child}`, return `false` (no change needed).
    2.  Compute valid voxel range for the parent dimension:
        *   $\mathbf{voxelMax} = (\mathbf{mVoxel} + (mDim - 1)) \ \& \ \sim(dim_{child} - 1)$
        *   $\mathbf{voxelMin} = \mathbf{mVoxel} \ \& \ \sim(dim_{child} - 1)$
    3.  Set `mDim = dim_{child}`.
    4.  Compute new position $\mathbf{p} = \mathbf{o} + mT0 \mathbf{d}$.
    5.  Compute new aligned voxel: $\mathbf{mVoxel} = \lfloor \mathbf{p} \rfloor \ \& \ \sim(dim_{child} - 1)$.
    6.  Clamp $\mathbf{mVoxel}$ to $[\mathbf{voxelMin}, \mathbf{voxelMax}]$.
    7.  For each axis $a$ where `mStep[a] != 0`:
        *   `mNext[a] = mT0 + (mVoxel[a] - p[a]) \times i[a]`
        *   If `mStep[a] > 0`, `mNext[a] += dim_{child} \times i[a]`.
    8.  Return `true`.

4.  **Hierarchical Traversal Logic:**
    *   Start at root level (e.g., $32^3$ or $16^3$ nodes).
    *   Initialize `NodeDDA` for the current level with `dim` and `child_size`.
    *   Loop:
        1.  Query `NodeDDA` to get local coordinates and child index.
        2.  Check if child is inside bounds and active (mask bit set).
        3.  If active:
            *   If child is a leaf: Perform leaf traversal or return intersection.
            *   If child is internal: Recursively call HDDA for the child level, updating `mDim` and `mT0` [S5, unofficial].
        4.  Step `NodeDDA` to next child.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hdda_init(ray, t_min, t_max, dim):
    """
    Initialize HDDA state.
    ray: object with .origin(), .dir(), .invDir(), .operator(t)
    """
    mDim = dim
    mT0 = t_min
    mT1 = t_max
    pos = ray(t_min)
    dir = ray.dir()
    inv = ray.invDir()
    
    # Align voxel origin to grid
    mVoxel = floor(pos) & ~(dim - 1)
    
    mStep = [0, 0, 0]
    mDelta = [0.0, 0.0, 0.0]
    mNext = [0.0, 0.0, 0.0]
    
    for axis in range(3):
        if dir[axis] == 0.0:
            mStep[axis] = 0
            mNext[axis] = float('inf')
        elif inv[axis] > 0:
            mStep[axis] = 1
            mNext[axis] = mT0 + (mVoxel[axis] + dim - pos[axis]) * inv[axis]
            mDelta[axis] = inv[axis]
        else:
            mStep[axis] = -1
            mNext[axis] = mT0 + (mVoxel[axis] - pos[axis]) * inv[axis]
            mDelta[axis] = -inv[axis]
            
    return {
        'mDim': mDim, 'mT0': mT0, 'mT1': mT1,
        'mVoxel': mVoxel, 'mStep': mStep,
        'mDelta': mDelta, 'mNext': mNext
    }

def hdda_step(state):
    """
    Advance HDDA to next voxel/node.
    Returns True if within bounds, False otherwise.
    """
    mDim = state['mDim']
    mT0 = state['mT0']
    mT1 = state['mT1']
    mVoxel = state['mVoxel']
    mStep = state['mStep']
    mDelta = state['mDelta']
    mNext = state['mNext']
    
    # Find axis with smallest next hit
    axis = argmin(mNext)
    
    # Round-off correction to prevent infinite loops/backtracking
    if mNext[axis] <= mT0:
        mNext[axis] += mT0 - 0.999999 * mNext[axis] + 1.0e-6
        
    mT0 = mNext[axis]
    mNext[axis] += mDim * mDelta[axis]
    mVoxel[axis] += mDim * mStep[axis]
    
    state['mT0'] = mT0
    state['mNext'] = mNext
    state['mVoxel'] = mVoxel
    
    return mT0 <= mT1

def hdda_update(state, ray, dim):
    """
    Update HDDA for a new dimension (child level).
    """
    if state['mDim'] == dim:
        return False
        
    mDim = state['mDim']
    mT0 = state['mT0']
    mVoxel = state['mVoxel']
    mStep = state['mStep']
    mNext = state['mNext']
    mDelta = state['mDelta']
    
    # Compute valid range for new dim
    voxelMax = (mVoxel + (mDim - 1)) & ~(dim - 1)
    voxelMin = mVoxel & ~(dim - 1)
    
    mDim = dim
    pos = ray(mT0)
    inv = ray.invDir()
    
    mVoxel = floor(pos) & ~(dim - 1)
    
    # Clamp to valid range
    for axis in range(3):
        mVoxel[axis] = max(voxelMin[axis], min(voxelMax[axis], mVoxel[axis]))
        
    for axis in range(3):
        if mStep[axis] == 0:
            continue
        mNext[axis] = mT0 + (mVoxel[axis] - pos[axis]) * inv[axis]
        if mStep[axis] > 0:
            mNext[axis] += dim * inv[axis]
            
    state['mDim'] = mDim
    state['mVoxel'] = mVoxel
    state['mNext'] = mNext
    
    return True
```

**Parameters:**
*   `ENFORCE_FORWARD_STEPPING`: Macro/flag to enable round-off correction [S3, unofficial].
*   `dim`: Integer dimension size for the current tree level. $Log2Dim=0$ is a voxel, $Log2Dim>0$ is a node [S6, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Direction:** If $d[axis] == 0$, the axis is disabled (`mNext` = $\infty$, `mStep` = 0) to prevent division by zero [S6, unofficial].
*   **Round-off Error:** Floating point inaccuracies can cause `mNext[axis] <= mT0`, leading to infinite loops. The correction term `mNext[axis] += mT0 - 0.999999f * mNext[axis] + 1.0e-6f` is required [S3, S6, unofficial].
*   **Non-linear Transforms:** HDDA assumes rays are linear in index space. For grids with non-linear transforms (e.g., frustum warping), an auxiliary binary VDB with a linear transform is used as a mask [S1, unofficial][S4].

## Subtopics

### NanoVolumeHDDA
NanoVolumeHDDA is an iterator that traverses a `NanoGrid` along a ray to yield merged active spans $(t_0, t_1)$ without heap allocations, using a callback interface [S1, unofficial]. It merges adjacent active leaves and tiles to eliminate seam artifacts at 8-voxel tile edges [S1, unofficial]. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by operating specifically on the `NanoGrid` structure and optimizing for zero-allocation iteration [S1, unofficial].

### [RayMajorantIterator](5199-raymajorantiterator.md)
The `RayMajorantIterator` is an iterator that traverses a sparse volume grid using a Hybrid Digital Differential Analyzer (HDDA) to step through voxels or leaves, computing the maximum density (majorant) for each segment where the majorant remains constant. It solves the problem of efficiently determining integration segments for volume path tracing by avoiding per-voxel sampling in empty or uniform regions. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by bundling consecutive steps with identical majorants into single segments, reducing overhead compared to standard voxel DDA which cannot skip inactive nodes or handle variable leaf sizes efficiently.

### [HDDA Hybrid](5201-hdda-hybrid.md)
HDDA Hybrid is a ray traversal algorithm for sparse voxel grids that combines hierarchical skipping with fine-grained traversal. It uses Hierarchical Digital Differential Analyzer (HDDA) to skip empty tiles at resolutions coarser than the leaf level, and switches to standard Digital Differential Analyzer (DDA) to traverse occupied tiles voxel-by-voxel. This approach avoids per-voxel checks in empty space while maintaining sampling accuracy in dense regions. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md).

### HDDA Branch
A ray traversal algorithm using HDDA that samples every tile/voxel, branching based on occupancy [S8, unofficial]. It initializes ray bounds and HDDA, then loops: if the current voxel is active, it records samples and advances; if not, it skips time. It steps the HDDA and updates the dimension dynamically based on the accessor [S8, unofficial].

### Distance LOD
Reduces traversal granularity for distant objects to save performance while maintaining surface color [S9, unofficial]. Beyond 400 voxels, the DDA terminates at brick granularity, and shading uses the brick's topmost solid material to keep distant terrain surface color [S9, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of active nodes/voxels intersected by the ray, but with a significantly smaller constant factor than standard DDA due to empty space skipping [S1, unofficial][S4].
*   **Memory Complexity:** $O(1)$ auxiliary memory for the DDA state, plus the grid storage [S1, unofficial].
*   **Performance:** Tested on RTX 3080 rendering WDAS cloud dataset at 1920x1080 in 6 ms [S7, unofficial].
*   **Strengths:** Highly efficient for sparse volumes; leverages compile-time template meta-programming to reduce overhead [S1, unofficial]; handles multi-resolution leapfrogging [S1, unofficial].
*   **Limitations:** Requires linear transforms for direct use; non-linear transforms require auxiliary grids [S1, unofficial][S4]; precision issues require round-off correction [S3, unofficial].
*   **When to use:** Sparse volume rendering, level set intersection, grids with linear transforms [S1, unofficial][S4][S7, unofficial].
*   **When not to use:** Dense volumes where skipping provides little benefit; grids with non-linear transforms without auxiliary mask generation [S1, unofficial][S4].

## Background
The HDDA was introduced by Ken Museth in the context of the VDB (Volume Database) structure, specifically for OpenVDB [S1, unofficial][S4]. It was developed to accelerate ray-marching through sparse volumes and narrow-band level sets, taking advantage of the B+tree structure of VDB [S1, unofficial]. The algorithm uses template meta-programming to inline the hierarchical traversal, reducing the overhead of vertical tree navigation [S1, unofficial]. It is implemented in OpenVDB v2 and later versions, and has been adapted for NanoVDB for GPU rendering [S3, S7, unofficial].

## Sources
[S1] Hierarchical Digital Differential Analyzer, https://museth.org/Ken/Publications_files/Museth_SIG14.pdf, fetched 2026-10-24 (unofficial)
[S2] NemoInfo/woxel, https://github.com/NemoInfo/woxel, fetched 2026-10-24 (unofficial)
[S3] openvdb/nanovdb/nanovdb/math/HDDA.h at master, https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/math/HDDA.h, fetched 2026-10-25 (unofficial)
[S4] Hierarchical digital differential analyzer for efficient ray-marching ..., https://www.semanticscholar.org/paper/7dbb75bb242819dc2ddae1a32c38849fb26742ad, fetched 2026-10-26
[S5] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S6] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S7] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S8] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S9] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
