# Hierarchical Digital Differential Analyzer (HDDA)

Also called: Device-Accelerated Raytracing, HDDA (Hierarchical Digital Differential Analyzer), HDDA ray marching, Hierarchical DDA (HDDA), Multi-level DDA, hierarchy of Digital Differential Analyzers

## Overview

The Hierarchical Digital Differential Analyzer (HDDA) is a ray-traversal algorithm that accelerates the marching of rays through sparse volumetric data structures, specifically OpenVDB and NanoVDB grids, by leveraging their hierarchical tree topology [S1, unofficial][S4]. It solves the problem of inefficient empty-space skipping in sparse volumes, where standard DDA advances one voxel at a time, by performing multi-resolution leapfrogging across tree levels (root, internal nodes, leaves) to skip large empty subspaces in single operations [S3, unofficial][S11]. Unlike standard DDA which uses a fixed step size, HDDA adapts its step size to the current resolution of the tree node being traversed, allowing it to jump over empty tiles and nodes [S5, unofficial][S6]. It is primarily used for volume rendering of densities (e.g., clouds) and ray-level-set intersection for narrow-band surfaces [S1, unofficial][S7].

## How it works

### Inputs and Outputs
*   **Input**: A ray defined by origin $\mathbf{o}$, direction $\mathbf{d}$, inverse direction $\mathbf{i} = 1/\mathbf{d}$, and time interval $[t_{min}, t_{max}]$; a VDB/NanoVDB grid with a fixed compile-time hierarchy; the current tree level dimension $dim$ (power of 2) [S11].
*   **Output**: A sequence of voxel or node indices $\mathbf{ijk}$ and corresponding ray parameters $t$ for occupied regions, or a traversal state that indicates when to switch to a finer/coarser level [S5, unofficial].

### Data Structures

The core state is maintained in the `HDDA` class [S5, unofficial].

**HDDA State Fields** [S5, unofficial][S9, unofficial]:
*   `mDim` (`int32_t`): The dimension size of the current tree level (e.g., 1 for leaf, 8 for lower internal, 128 for upper, 4096 for root children) [S9, unofficial].
*   `mT0` (`RealT`): Current ray parameter (time) at the start of the current step.
*   `mT1` (`RealT`): Maximum ray parameter (end of ray).
*   `mVoxel` (`CoordT`): Current voxel/node index (aligned to `mDim`).
*   `mStep` (`CoordT`): Step direction per axis (+1, -1, or 0).
*   `mDelta` (`Vec3T`): Time delta per step per axis ($\Delta t$).
*   `mNext` (`Vec3T`): Next hit time per axis.

**NodeDDA State Fields** (for internal node traversal) [S8, unofficial]:
*   `state`: Underlying DDA with `voxel_size = child_size`.
*   `origin` (`Coord`): Node origin in index space.
*   `dim` (`Int32`): Children per axis (e.g., 8, 16, 32).
*   `child_size` (`Int32`): Child coverage in voxels (e.g., 1, 8, 128).
*   `log2cs` (`Int`): $\log_2(\text{child\_size})$ for bit-shift index computation.

**VDB Leaf Structure** (example from VDB345) [S2, unofficial]:
*   `color`: `vec3<f32>`
*   `dist`: `u32` (distance to nearest non-empty node/surface)
*   `num_parents`: `u32` (hierarchy level 0-3)
*   `parents`: `array<Parent, 3>` (Parent: `origin: vec3<i32>`, `idx: u32`)

### Initialization

The algorithm initializes the DDA state for a specific dimension $dim$ [S5, unofficial][S9, unofficial].

1.  Set `mDim = dim`, `mT0 = startTime`, `mT1 = maxTime` [S5, unofficial].
2.  Get current ray position $\mathbf{pos} = \text{ray}(mT0)$ and inverse direction $\mathbf{i}$ [S5, unofficial].
3.  Compute aligned voxel index: $\mathbf{mVoxel} = \lfloor \mathbf{pos} \rfloor \ \& \ (\sim(dim - 1))$ [S5, unofficial].
4.  For each axis $a \in \{0, 1, 2\}$:
    *   If $\mathbf{d}[a] == 0$:
        *   Set `mNext[a] = Maximum<RealT>::value()` (disabled) [S5, unofficial].
        *   Set `mStep[a] = 0` [S5, unofficial].
    *   Else if $\mathbf{i}[a] > 0$:
        *   Set `mStep[a] = 1` [S5, unofficial].
        *   Set `mNext[a] = mT0 + (mVoxel[a] + dim - pos[a]) * i[a]` [S5, unofficial].
        *   Set `mDelta[a] = i[a]` [S5, unofficial].
    *   Else ($\mathbf{i}[a] < 0$):
        *   Set `mStep[a] = -1` [S5, unofficial].
        *   Set `mNext[a] = mT0 + (mVoxel[a] - pos[a]) * i[a]` [S5, unofficial].
        *   Set `mDelta[a] = -i[a]` [S5, unofficial].

### Step Function

Advances the ray to the next intersected voxel or node boundary [S5, unofficial][S9, unofficial].

1.  Find axis $a = \text{MinIndex}(\mathbf{mNext})$ [S5, unofficial][S9, unofficial].
2.  **Round-off Correction** (if `ENFORCE_FORWARD_STEPPING` is defined):
    *   If `mNext[a] <= mT0`:
        *   `mNext[a] += mT0 - 0.999999f * mNext[a] + 1.0e-6f` [S5, unofficial][S9, unofficial].
3.  Update time: `mT0 = mNext[a]` [S5, unofficial].
4.  Update next hit time: `mNext[a] += mDim * mDelta[a]` [S5, unofficial].
5.  Update voxel index: `mVoxel[a] += mDim * mStep[a]` [S5, unofficial].
6.  Return `mT0 <= mT1` [S5, unofficial].

### Update Function (Level Switching)

When the ray enters a region requiring a different resolution (e.g., entering a finer leaf node from a coarser internal node), the state is re-initialized [S5, unofficial][S9, unofficial].

1.  If `mDim == dim`, return `false` (no change needed) [S5, unofficial].
2.  Compute valid voxel range for the transition:
    *   $\mathbf{voxelMax} = (\mathbf{mVoxel} + \text{Coord}(mDim - 1)) \ \& \ (\sim(dim - 1))$ [S5, unofficial].
    *   $\mathbf{voxelMin} = \mathbf{mVoxel} \ \& \ (\sim(dim - 1))$ [S5, unofficial].
3.  Set `mDim = dim` [S5, unofficial].
4.  Get current position $\mathbf{pos} = \text{ray}(mT0)$ and inverse direction $\mathbf{i}$ [S5, unofficial].
5.  Compute new aligned voxel: $\mathbf{mVoxel} = \lfloor \mathbf{pos} \rfloor \ \& \ (\sim(dim - 1))$ [S5, unofficial].
6.  **Clamp** $\mathbf{mVoxel}$ to $[\mathbf{voxelMin}, \mathbf{voxelMax}]$ to prevent out-of-bounds access [G].
7.  For each axis $a$ where `mStep[a] != 0`:
    *   `mNext[a] = mT0 + (mVoxel[a] - pos[a]) * i[a]` [S5, unofficial].
    *   If `mStep[a] > 0`, `mNext[a] += dim * i[a]` [S5, unofficial].
8.  Return `true` [S5, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class HDDA:
    def __init__(self):
        self.mDim = 0
        self.mT0 = 0.0
        self.mT1 = 0.0
        self.mVoxel = [0, 0, 0]
        self.mStep = [0, 0, 0]
        self.mDelta = [0.0, 0.0, 0.0]
        self.mNext = [0.0, 0.0, 0.0]

    def init(self, ray_origin, ray_dir, ray_inv_dir, start_time, max_time, dim):
        """Initialize HDDA for a specific dimension."""
        self.mDim = dim
        self.mT0 = start_time
        self.mT1 = max_time
        
        # Compute position at mT0
        pos = [ray_origin[i] + start_time * ray_dir[i] for i in range(3)]
        
        # Align voxel index to current dimension
        self.mVoxel = [math.floor(pos[i]) & ~(dim - 1) for i in range(3)]
        
        for axis in range(3):
            inv = ray_inv_dir[axis]
            if inv == 0.0:
                self.mNext[axis] = float('inf')
                self.mStep[axis] = 0
            elif inv > 0.0:
                self.mStep[axis] = 1
                self.mNext[axis] = self.mT0 + (self.mVoxel[axis] + dim - pos[axis]) * inv
                self.mDelta[axis] = inv
            else:
                self.mStep[axis] = -1
                self.mNext[axis] = self.mT0 + (self.mVoxel[axis] - pos[axis]) * inv
                self.mDelta[axis] = -inv

    def step(self):
        """Advance to next voxel/node. Returns True if within bounds."""
        min_val = float('inf')
        axis = 0
        for i in range(3):
            if self.mNext[i] < min_val:
                min_val = self.mNext[i]
                axis = i
        
        # Round-off correction to prevent backward stepping or stalling
        if self.mNext[axis] <= self.mT0:
            self.mNext[axis] += self.mT0 - 0.999999 * self.mNext[axis] + 1.0e-6
            
        self.mT0 = self.mNext[axis]
        self.mNext[axis] += self.mDim * self.mDelta[axis]
        self.mVoxel[axis] += self.mDim * self.mStep[axis]
        
        return self.mT0 <= self.mT1

    def update(self, ray_origin, ray_dir, ray_inv_dir, dim):
        """Update state for a new dimension (level switch)."""
        if self.mDim == dim:
            return False
            
        # Compute valid range for clamping
        voxel_max = [(self.mVoxel[i] + (self.mDim - 1)) & ~(dim - 1) for i in range(3)]
        voxel_min = [self.mVoxel[i] & ~(dim - 1) for i in range(3)]
        
        self.mDim = dim
        # Compute position at current mT0
        pos = [ray_origin[i] + self.mT0 * ray_dir[i] for i in range(3)]
                
        self.mVoxel = [math.floor(pos[i]) & ~(dim - 1) for i in range(3)]
        
        # Clamp to valid range
        for i in range(3):
            self.mVoxel[i] = max(voxel_min[i], min(voxel_max[i], self.mVoxel[i]))
            
        for axis in range(3):
            if self.mStep[axis] == 0:
                continue
            inv = ray_inv_dir[axis]
            self.mNext[axis] = self.mT0 + (self.mVoxel[axis] - pos[axis]) * inv
            if self.mStep[axis] > 0:
                self.mNext[axis] += dim * inv
                
        return True
```

### Parameters and Typical Values

*   **VDB Hierarchy Levels** [S11]:
    *   Root children: $4096^3$ (dim=4096).
    *   Upper internal: $128^3$ (dim=128).
    *   Lower internal: $8^3$ (dim=8).
    *   Leaf: $1^3$ (dim=1).
*   **ENFORCE_FORWARD_STEPPING**: Macro to enable round-off correction. Default: Enabled [S5, unofficial].
*   **HDDA_MAX_RAY_STEPS**: Maximum iterations. Typical: 1000 [S2, unofficial].
*   **MIN_DENSITY**: Threshold for skipping empty space in rendering. Typical: 0.01 [S10, unofficial].
*   **step_size**: Fixed step size for sampling. Typical: 0.57 [S10, unofficial].

### Edge Cases and Pitfalls

*   **Zero Direction Components**: If $\mathbf{d}[a] == 0$, the axis is disabled (`mNext[a] = \infty`, `mStep[a] = 0`) to prevent division by zero [S9, unofficial].
*   **Round-off Errors**: Floating-point inaccuracies can cause `mNext[a] <= mT0`, leading to infinite loops or backward stepping. The correction term `mT0 - 0.999999f * mNext[a] + 1.0e-6f` forces forward progress [S5, unofficial][S9, unofficial].
*   **Non-linear Transforms**: HDDA assumes rays are linear in index space [S1, unofficial]. For grids warped to a camera frustum, shadow rays may not be linear [S1, unofficial]. Solution: Construct an auxiliary binary VDB with a linear transform to act as a mask [S1, unofficial][S7].
*   **Self-Intersection**: A small offset (`4e-4 * step * mask`) is added to the position after each step to prevent hitting the same voxel [S2, unofficial].
*   **Out of Bounds**: Check if `abs(p) > 4096` (or grid bounds) to handle rays escaping the volume [S2, unofficial].

## Subtopics

### [OpenVDB DDA](2149-openvdb-dda.md)
The OpenVDB Digital Differential Analyzer (DDA) is a fixed-step ray traversal algorithm designed to efficiently iterate through the hierarchical node and voxel structure of OpenVDB grids. It solves the problem of determining which grid elements (tiles, internal nodes, or voxels) intersect a given ray by adapting Bresenham’s line algorithm to 3D ray-space. Unlike its sibling Hierarchical Digital Differential Analyzer (HDDA), which uses variable step sizes to leapfrog across tree levels, OpenVDB DDA uses a fixed step size defined by the template parameter `Log2Dim` (or `Dim` in NanoVDB), making it suitable for uniform traversal of specific tree levels or leaf voxels. It is primarily used as the core traversal kernel within the broader HDDA framework to handle leaf-level or specific internal-node-level intersections.

### NanoVolumeHDDA
NanoVolumeHDDA is an iterator that traverses a `NanoGrid` along a ray to yield merged active spans $(t_0, t_1)$ without heap allocations, using a callback interface [S1, unofficial]. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by operating specifically on the `NanoGrid` structure and optimizing for zero-allocation iteration [S1, unofficial].

### [RayMajorantIterator](5199-raymajorantiterator.md)
The `RayMajorantIterator` is an iterator that traverses a sparse volume grid using a Hybrid Digital Differential Analyzer (HDDA) to step through voxels or leaves, computing the maximum density (majorant) for each segment where the majorant remains constant. It solves the problem of efficiently determining integration segments for volume path tracing by avoiding per-voxel sampling in empty or uniform regions. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by bundling consecutive steps with identical majorants into single segments, reducing overhead compared to standard voxel DDA which cannot skip inactive nodes or handle variable leaf sizes efficiently.

### [HDDA Hybrid](5201-hdda-hybrid.md)
HDDA Hybrid is a ray traversal algorithm for sparse voxel grids that combines hierarchical skipping with fine-grained traversal. It uses Hierarchical Digital Differential Analyzer (HDDA) to skip empty tiles at resolutions coarser than the leaf level, and switches to standard Digital Differential Analyzer (DDA) to traverse occupied tiles voxel-by-voxel. This approach avoids per-voxel checks in empty space while maintaining sampling accuracy in dense regions. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md).

### HDDA Branch
A ray traversal algorithm using HDDA that samples every tile/voxel, branching based on occupancy [S12, unofficial]. It initializes ray bounds and HDDA, then loops: if the current voxel is active, it records samples and advances time; if not active, it skips time [S12, unofficial]. It steps the HDDA and updates the dimension if necessary [S12, unofficial].

### Teardown Hierarchical Traversal
A hierarchical traversal method that uses DDA within a level and switches levels by rescaling the grid so the new level becomes a level 0 grid, restarting the DDA algorithm [S13, unofficial]. It calculates ray position using $pos = \lfloor (O + t \cdot D) / 2^{level} \rfloor$ [S13, unofficial]. Pitfalls include floating-point issues causing outliers in depth/normal buffers and rays traversing corners causing normal issues [S13, unofficial].

### Distance LOD
Reduces traversal granularity for distant objects to save performance while maintaining surface color [S14, unofficial]. Beyond 400 voxels, the DDA terminates at brick granularity, and shading uses the brick's topmost solid material [S14, unofficial].

## Operating characteristics

*   **Time Complexity**: $O(N)$ where $N$ is the number of occupied voxels/nodes visited [S11]. Significantly faster than standard DDA ($O(V)$ where $V$ is total voxels) in sparse volumes because it skips empty space [S3, unofficial][S11].
*   **Memory Complexity**: Low overhead. HDDA uses 1.5x to 3x less memory than DDA in dense bitfields and over 100x less than NerfAcc at high resolutions (e.g., 8.85 MB vs 1028 MB at $1024^3$) [S11].
*   **Performance**: On a 3-voxel-wide Stanford bunny shell, HDDA processes 3.77M rays/sec at $32^3$ resolution vs 2.57M for NerfAcc, and 1.43M rays/sec at $1024^3$ vs 0.47M for NerfAcc [S11].
*   **Strengths**: Extremely efficient for sparse volumes with large empty regions; minimizes interpolation cost for level sets; compiles to high-performance code via template meta-programming [S1, unofficial][S4].
*   **Limitations**: Requires VDB/NanoVDB data structure; less effective on dense grids or non-linear transforms without auxiliary masks; may not outperform standard DDA on datasets like Mip-NeRF 360 where occupancy is high [S3, unofficial].
*   **When to Use**: Sparse volumes (clouds, level sets), real-time rendering on GPU (e.g., Unity HDRP), differentiable rendering (fVDB) [S10, unofficial][S11].
*   **When Not to Use**: Dense grids, non-linear transforms without auxiliary mask, or when memory overhead of VDB structure is prohibitive [S3, unofficial][S7].

## Background

The Hierarchical Digital Differential Analyzer was introduced by Ken Museth in the SIGGRAPH 2014 paper "Hierarchical Digital Differential Analyzer for Efficient Ray-Marching in OpenVDB" [S4]. It was developed to accelerate ray-tracing of narrow-band level set surfaces and volume rendering (e.g., clouds) within the OpenVDB data structure [S1, unofficial][S7]. The algorithm leverages the B+tree structure of VDB, which has a fixed compile-time configuration, allowing template meta-programming to inline and collapse the hierarchical traversal, reducing vertical navigation overhead [S1, unofficial][S4]. It has since been adopted in various graphics engines and frameworks, including Unity HDRP, fVDB, and woxel [S10, unofficial][S11][S2, unofficial].

## Sources

[S1] Hierarchical Digital Differential Analyzer, https://museth.org/Ken/Publications_files/Museth_SIG14.pdf, fetched 2026-10-24 (unofficial)
[S2] NemoInfo/woxel, https://github.com/NemoInfo/woxel, fetched 2026-10-24 (unofficial)
[S3] Plug-and-Play Acceleration of Occupancy Grid-based NeRF Rendering using VDB Grid and Hierarchical Ray Traversal, https://doi.org/10.48550/arxiv.2404.10272, fetched 2026-10-24 (unofficial)
[S4] Hierarchical digital differential analyzer for efficient ray-marching in OpenVDB, https://doi.org/10.1145/2614106.2614136, fetched 2026-11-08
[S5] openvdb/nanovdb/nanovdb/math/HDDA.h at master, https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/math/HDDA.h, fetched 2026-10-25 (unofficial)
[S6] nanovdb/util/HDDA.h File Reference - OpenVDB, https://academysoftwarefoundation.github.io/openvdb/HDDA_8h.html, fetched 2026-10-25
[S7] Hierarchical digital differential analyzer for efficient ray-marching ..., https://www.semanticscholar.org/paper/7dbb75bb242819dc2ddae1a32c38849fb26742ad, fetched 2026-10-26
[S8] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S9] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S10] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
[S11] fVDB: A Deep-Learning Framework for Sparse, Large-Scale, and High-Performance Spatial Intelligence, https://research.nvidia.com/labs/prl/publication/williams2024fvdb/, fetched 2026-10-30
[S12] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S13] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S14] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
