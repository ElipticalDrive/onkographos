# Zero-Crossing DDA Traversal

Also called: Zero Crossing Detection, ZeroCrossing, ZeroCrossingNode

## Overview
Zero-Crossing DDA Traversal performs empty-space ray-marching at all levels of a hierarchical grid using a Hierarchical Digital Differential Analyzer (HDDA) to detect the first voxel where a scalar field changes sign relative to the entry voxel's value [S1, unofficial][S2]. It solves the problem of efficiently skipping empty or uniform regions by combining coarse hierarchical traversal with narrow-band voxel-level detection [S1, unofficial][S2]. Unlike standard DDA methods that visit every voxel, this method returns immediately upon detecting a sign change, making it suitable for isosurface extraction or narrow-band volume rendering [S1, unofficial][S2]. It builds on [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) and differs from [DDA](2323-dda.md), [Two-Step Discrete Ray Tracing](5204-two-step-discrete-ray-tracing.md), [Voxel Occupancy Mask](5242-voxel-occupancy-mask.md), [Parallel 3D-DDA](6247-parallel-3d-dda.md), and [Nearest Neighbor Ray Traversal](8321-nearest-neighbor-ray-traversal.md) by halting traversal at the first sign change rather than sampling all active voxels.

## How it works
### Inputs and Outputs
- **Inputs**: `ray` (object with `clip(bbox)`, `t0()`, `t1()`, `invDir()`, and `operator()(time)`), `acc` (accessor with `root().bbox()`, `getDim(ijk, ray)`, `isActive(ijk)`, and `getValue(ijk)`), `ijk` (coordinate reference), `v` (value reference), `t` (time reference) [S1, unofficial][S2].
- **Outputs**: Boolean indicating success; `ijk`, `v`, and `t` updated on success [S1, unofficial][S2].

### Data Structures
- **HDDA**: State machine for hierarchical stepping. Maintains `mDelta` (time delta to next voxel boundary) and `mNext` (next voxel coordinates) [S1, unofficial]. Provides `step()` to advance, `time()` to get current ray parameter, `voxel()` to get current voxel index, `dim()` to get current node dimension, `init()` to initialize, and `update()` to reinitialize with a new dimension [S1, unofficial].
- **Coord**: Integer coordinate type. `RoundDown<Coord>(pos)` or `Coord::Floor(pos)` computes the voxel index containing a 3D position [S1, unofficial][S2].

### Parameters
- **Delta**: Constant `1.0001` used to offset the ray position when determining the next voxel index after a step, ensuring the next voxel is correctly identified and avoiding boundary precision issues [S1, unofficial].

### Algorithm Steps
1. **Ray Clipping**: Clip the ray to the root bounding box. If clipping fails or `ray.t1() > 1e20`, return `false` [S1, unofficial][S2].
2. **Initialization**: Set `Delta = 1.0001`. Compute initial `ijk = RoundDown<Coord>(ray.start())`. Initialize `HDDA` with `ray` and `acc.getDim(ijk, ray)`. Read `v0 = acc.getValue(ijk)` [S1, unofficial].
3. **Hierarchical Traversal**: While `hdda.step()` returns true:
   - Compute `ijk = RoundDown<Coord>(ray(hdda.time() + Delta))`.
   - Update `hdda.update(ray, acc.getDim(ijk, ray))`.
   - If `hdda.dim() > 1` (tile value) or `!acc.isActive(ijk)` (inactive voxel), continue to the next step [S1, unofficial].
4. **Narrow-Band Traversal**: While `hdda.step()` returns true and `acc.isActive(hdda.voxel())` is true:
   - Read `v = acc.getValue(hdda.voxel())`.
   - Check for zero-crossing: If `v * v0 < 0`:
     - Set `ijk = hdda.voxel()`.
     - Set `t = hdda.time()`.
     - Return `true` [S1, unofficial].
5. **Termination**: If the loops complete without finding a zero-crossing, return `false` [S1, unofficial][S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def zero_crossing_dda(ray, acc):
    # 1. Ray Clipping
    if not ray.clip(acc.root().bbox()) or ray.t1() > 1e20:
        return False, None, None, None

    Delta = 1.0001
    ijk = math.floor(ray.start())

    # 2. Initialize HDDA
    hdda = HDDA(ray, acc.get_dim(ijk, ray))
    v0 = acc.get_value(ijk)

    # 3. Hierarchical Traversal Loop
    while hdda.step():
        ijk = math.floor(ray(hdda.time() + Delta))
        hdda.update(ray, acc.get_dim(ijk, ray))

        # Skip tiles or inactive voxels
        if hdda.dim() > 1 or not acc.is_active(ijk):
            continue

        # 4. Narrow-Band Traversal Loop
        while hdda.step() and acc.is_active(hdda.voxel()):
            v = acc.get_value(hdda.voxel())
            if v * v0 < 0:
                ijk = hdda.voxel()
                t = hdda.time()
                return True, ijk, v, t

    return False, None, None, None
```

### Edge Cases and Pitfalls
- **Precision**: The `Delta` offset of `1.0001` prevents sampling the same voxel twice due to floating-point inaccuracies at voxel boundaries [S1, unofficial].
- **Inactive Voxels & Tiles**: The algorithm explicitly skips inactive voxels and hierarchical tiles (`dim() > 1`) to ensure voxel-level precision for the zero-crossing check [S1, unofficial].
- **Fixed Reference Value**: The sign change is always compared against `v0` (the value at the entry voxel), not the previous voxel's value, meaning it only detects the first crossing relative to the ray origin's sign [S1, unofficial].
- **No Zero-Crossing**: If the ray passes through a region where the sign never changes relative to `v0`, the function returns `false` [S1, unofficial][S2].

## Operating characteristics
- **Time Complexity**: Depends on the sparsity of the zero-crossing. In empty space, HDDA skips large volumes efficiently. In dense narrow bands, it behaves like standard DDA but stops at the first sign change [S1, unofficial].
- **Memory Complexity**: O(1) auxiliary memory, as it uses iterative stepping without recursion or large stacks [S1, unofficial].
- **Strengths**: Efficient for sparse volumes; avoids unnecessary voxel visits in empty regions; precise voxel-level localization [S1, unofficial].
- **Limitations**: Only detects the first zero-crossing relative to the entry point value; may miss multiple zero-crossings in a single ray pass; requires active voxel data [S1, unofficial].

## Background
The method is implemented in the NanoVDB library as part of the OpenVDB ecosystem, which is widely used in visual effects and scientific visualization for large-scale volumetric data [S1, unofficial][S2]. The HDDA class is designed for hierarchical grids, allowing efficient traversal of multi-resolution data structures [S1, unofficial].

## Sources
[S1] openvdb/nanovdb/nanovdb/math/HDDA.h at master, https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/math/HDDA.h, fetched 2026-10-25 (unofficial)
[S2] nanovdb/util/HDDA.h File Reference - OpenVDB, https://academysoftwarefoundation.github.io/openvdb/HDDA_8h.html, fetched 2026-10-25
[S3] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
