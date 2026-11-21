# Bolt

Also called: Bypass expensive BVH query for exterior voxels

## Overview
Bolt is an optimization for solid voxelization that skips the expensive Boundary Volume Hierarchy (BVH) distance query for voxels determined to be exterior to the mesh, assigning them a placeholder infinite distance instead [S1, S2, unofficial]. It sets itself apart from standard solid voxelization by avoiding the computation of the exact Signed Distance Field (SDF) for points that are guaranteed to be discarded by the interiority filter, provided no narrow band is required [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A voxel center point $P$, a boolean `is_inside` indicating if $P$ is inside the mesh (derived from a parity check), and an optional parameter `narrow_band` [S1, S2, unofficial].
- Output: The SDF value $d$ for the voxel center [S1, S2, unofficial].

**Data Structures:**
- `narrow_band`: An optional flag. When `None`, the optimization is active [S1, S2, unofficial]. When `Some`, the BVH query is performed regardless of interior/exterior status [S1, S2, unofficial].
- `is_inside`: A boolean result from a parity check that determines if a voxel is inside the mesh [S1, S2, unofficial].

**Steps:**
1. For each voxel center $P$, determine if it is inside the mesh (`is_inside`) and if a narrow band is requested (`narrow_band`) [S2, unofficial].
2. If `narrow_band` is `None` (disabled) AND the voxel is exterior (`!is_inside`), assign the distance $d = \text{f32::INFINITY}$ [S1, S2, unofficial].
3. Otherwise (if `narrow_band` is specified OR the voxel is interior), compute the exact distance using the mesh's BVH query: $d = \text{self.mesh.distance\_to\_local\_point}(P, \text{false})$ cast to `f32` [S1, S2, unofficial].
4. In solid voxelization mode (`surface_only = false`), apply the filter $d \le 0.0$ to discard exterior voxels [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sdf_bolt(point_3d, is_inside, narrow_band):
    # If no narrow band is required and the point is outside,
    # skip the expensive BVH query and return infinity.
    # These points will be discarded by the sdf <= 0.0 filter anyway.
    if not is_inside and narrow_band is None:
        return float('inf')
    
    # Otherwise, compute the exact distance via BVH.
    # This handles interior voxels and exterior voxels when a narrow band is needed.
    exact_distance = self.mesh.distance_to_local_point(point_3d, false)
    return float(exact_distance)
```

**Parameters:**
- `narrow_band`: Optional. When `None`, the optimization is active. When `Some`, the BVH query is performed regardless of interior/exterior status [S1, S2, unofficial].
- `is_inside`: Boolean result from a parity check [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Misclassification:** Points exactly on the mesh surface may be classified as outside by the parity test if the implementation uses `partition_point(|&hx| hx <= x)`, which excludes hits at the same x coordinate [S1, S2, unofficial].
- **Data Loss:** Previously, these boundary samples received `distance == 0` and passed the `sdf <= 0.0` filter. With Bolt, they receive `+inf` and are dropped [S1, S2, unofficial].
- **Impact:** This can remove valid boundary voxels or cause small meshes to produce no solid particles on aligned/coarse grids [S1, S2, unofficial]. The optimization must ensure exact-surface voxels are retained or handled correctly [S2, unofficial].

## Operating characteristics
- **Performance:** Benchmarks show a performance improvement of ~3.5% - 5.2% on standard tests for solid voxelization [S2, unofficial]. Another benchmark measured a drop from ~32ms to ~30-31ms per loop over 100 sample passes [S1, unofficial].
- **Scaling:** The improvement is significant for high-resolution solid grids where a large percentage of sampled voxels reside outside the target mesh bounds [S1, S2, unofficial].
- **Strengths:** Drastically reduces processing time by avoiding unnecessary BVH queries for exterior points [S1, S2, unofficial].
- **Limitations:** Potential loss of boundary voxels if the parity test misclassifies surface points as exterior [S1, S2, unofficial].

## Background
Bolt was introduced as an optimization in the `voxelizer-rs` project to bypass expensive BVH queries for exterior voxels in solid voxelization mode when a narrow band is not specified [S1, S2, unofficial]. It was implemented by lhemerly and reviewed by Jules in June 2026 [S1, S2, unofficial].

## Sources
[S1] ⚡ Bolt: Bypass expensive BVH query for exterior voxels, https://github.com/lhemerly/voxelizer-rs/pull/188, fetched 2026-10-27 (unofficial)
[S2] ⚡ Bolt: Bypass BVH query for exterior voxels in solid voxelization, https://github.com/lhemerly/voxelizer-rs/pull/199, fetched 2026-10-27 (unofficial)
