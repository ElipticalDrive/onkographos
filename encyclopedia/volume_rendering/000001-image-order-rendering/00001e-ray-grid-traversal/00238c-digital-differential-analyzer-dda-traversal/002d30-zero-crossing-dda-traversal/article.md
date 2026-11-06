# Zero-Crossing DDA Traversal

Also called: Zero Crossing Detection, ZeroCrossing, ZeroCrossingNode

## Overview
Zero-Crossing DDA Traversal is a specialized ray-marching algorithm designed to locate the first point where a scalar field changes sign (a zero-crossing) within a hierarchical grid structure, such as OpenVDB. It solves the problem of efficiently skipping empty or uniform regions by combining hierarchical DDA (HDDA) for coarse traversal with narrow-band DDA for precise voxel-level detection. Unlike standard DDA methods that visit every voxel, this method returns immediately upon detecting a sign change between the current voxel value and the previous reference value, making it ideal for isosurface extraction or narrow-band volume rendering.

## How it works
The method operates on a hierarchical grid accessor and a ray object. It assumes the grid values are floating-point numbers.

### Inputs and Outputs
- **Inputs**:
  - `ray`: A ray object with methods `clip(bbox)`, `t0()`, `t1()`, `invDir()`, and `operator()(time)` returning the 3D position at a given time.
  - `acc`: An accessor object providing `root().bbox()`, `getDim(ijk, ray)`, `isActive(ijk)`, and `getValue(ijk)`.
  - `ijk`: Output coordinate variable (initialized to the first hit).
  - `v`: Output value variable (set to the value at the zero-crossing voxel).
  - `t`: Output time variable (set to the intersection time).
- **Outputs**:
  - Boolean `true` if a zero-crossing is found; `false` otherwise.
  - Updated `ijk`, `v`, and `t` if successful.

### Data Structures
- **HDDA (Hierarchical DDA)**: A state machine that steps through the grid hierarchy. It maintains `mDelta` (time delta to next voxel boundary) and `mNext` (next voxel coordinates). It provides `step()` to advance, `time()` to get current ray parameter, `voxel()` to get current voxel index, and `dim()` to get current node dimension.
- **Coord**: Integer coordinate type. `RoundDown<Coord>(pos)` or `Coord::Floor(pos)` computes the voxel index containing a 3D position.

### Parameters
- **Delta**: A small constant offset, typically `1.0001f`, used to ensure the ray position is strictly inside the next voxel after a step, avoiding boundary precision issues [S1, S2, unofficial].

### Algorithm Steps
1. **Ray Clipping**: Clip the ray to the root bounding box of the grid. If the ray does not intersect the bbox or `ray.t1()` exceeds a large threshold (e.g., `1e20`), return `false` [S1, S2, unofficial].
2. **Initialization**:
   - Set `Delta = 1.0001f` [S1, S2, unofficial].
   - Compute the initial voxel index `ijk` as the floor of the ray start position: `ijk = RoundDown<Coord>(ray.start())` [S1, unofficial].
   - Initialize the HDDA with the ray and the dimension at `ijk`: `hdda.init(ray, acc.getDim(ijk, ray))` [S1, unofficial].
   - Read the initial value `v0 = acc.getValue(ijk)` [S1, S2, unofficial].
3. **Hierarchical Traversal Loop**:
   - While `hdda.step()` returns true:
     - Compute the next voxel index using the offset: `ijk = RoundDown<Coord>(ray(hdda.time() + Delta))` [S1, unofficial].
     - Update the HDDA state with the new voxel's dimension: `hdda.update(ray, acc.getDim(ijk, ray))` [S1, unofficial].
     - If `hdda.dim() > 1` (tile value) or `!acc.isActive(ijk)` (inactive voxel), continue to the next step [S1, unofficial].
4. **Narrow-Band Traversal Loop**:
   - While `hdda.step()` returns true AND `acc.isActive(hdda.voxel())` is true:
     - Get the current voxel value `v = acc.getValue(hdda.voxel())` [S1, S2, unofficial].
     - Check for zero-crossing: If `v * v0 < 0`:
       - Set `ijk = hdda.voxel()` [S1, S2, unofficial].
       - Set `t = hdda.time()` [S1, S2, unofficial].
       - Set `v` (output) to the current value [S1, unofficial].
       - Return `true` [S1, S2, unofficial].
5. **Termination**: If the loops complete without finding a zero-crossing, return `false` [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zero_crossing_dda(ray, acc):
    # 1. Ray Clipping
    if not ray.clip(acc.root().bbox()):
        return False
    if ray.t1() > 1e20:
        return False

    Delta = 1.0001f
    ijk = floor(ray.start())
    
    # 2. Initialize HDDA
    hdda = HDDA()
    hdda.init(ray, acc.get_dim(ijk, ray))
    
    v0 = acc.get_value(ijk)
    
    # 3. Hierarchical Traversal
    while hdda.step():
        # Compute next voxel index with offset
        next_pos = ray(hdda.time() + Delta)
        ijk = floor(next_pos)
        
        # Update HDDA for new voxel's dimension
        hdda.update(ray, acc.get_dim(ijk, ray))
        
        # Skip tiles or inactive voxels
        if hdda.dim() > 1 or not acc.is_active(ijk):
            continue
            
        # 4. Narrow-Band Traversal
        while hdda.step() and acc.is_active(hdda.voxel()):
            v = acc.get_value(hdda.voxel())
            
            # Check for sign change
            if v * v0 < 0:
                ijk = hdda.voxel()
                t = hdda.time()
                return True, ijk, v, t
        
        # Update v0 for next hierarchical step? 
        # Note: Source [S1, unofficial] does not explicitly update v0 in the outer loop.
        # It uses the initial v0 for all comparisons. 
        # This implies the method detects the first zero crossing relative 
        # to the entry point value, or relies on the narrow band to be 
        # contiguous. However, standard zero-crossing detection usually 
        # compares adjacent voxels. Source [S1, unofficial] compares against v0.
        # Source [S2, unofficial] also compares against v0.
        # [G] In some implementations, v0 might be updated to v after 
        # each narrow-band step to detect local sign changes. 
        # Based strictly on [S1, unofficial], v0 is fixed.
        
    return False
```

### Edge Cases and Pitfalls
- **Precision**: The `Delta` offset is critical to avoid sampling the same voxel twice due to floating-point inaccuracies at voxel boundaries [S1, S2, unofficial].
- **Inactive Voxels**: The algorithm explicitly skips inactive voxels, which may represent empty space or background. If the zero-crossing lies within an inactive region, it will be missed unless the inactive region is part of the active narrow band [S1, unofficial].
- **Tile Values**: Hierarchical nodes (tiles) are skipped (`dim() > 1`) to ensure voxel-level precision for the zero-crossing check [S1, unofficial].
- **No Zero-Crossing**: If the ray passes through a region where the sign never changes relative to `v0`, the function returns `false` [S1, S2, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the sparsity of the zero-crossing. In empty space, HDDA skips large volumes efficiently. In dense narrow bands, it behaves like standard DDA but stops at the first sign change.
- **Memory Complexity**: O(1) auxiliary memory, as it uses iterative stepping without recursion or large stacks.
- **Strengths**: Efficient for sparse volumes; avoids unnecessary voxel visits in empty regions; precise voxel-level localization.
- **Limitations**: Only detects the first zero-crossing relative to the entry point value; may miss multiple zero-crossings in a single ray pass; requires active voxel data.

## Background
The Zero-Crossing DDA Traversal is implemented in the NanoVDB library, part of the OpenVDB ecosystem, which is widely used in visual effects and scientific visualization for large-scale volumetric data. The HDDA class is designed for hierarchical grids, allowing efficient traversal of multi-resolution data structures. The specific `zeroCrossing` function is used in applications like the HNanoViewer for interactive rendering of VDB data [S1, S2, unofficial].

## Sources
[S1] openvdb/nanovdb/nanovdb/math/HDDA.h at master, https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/math/HDDA.h, fetched 2026-10-25 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
