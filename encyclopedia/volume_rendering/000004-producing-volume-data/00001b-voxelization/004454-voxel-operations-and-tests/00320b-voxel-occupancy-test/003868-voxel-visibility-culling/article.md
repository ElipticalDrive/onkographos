# Voxel Visibility Culling

Also called: Inside/Outside Clipping

## Overview
Voxel Visibility Culling is a post-processing step in voxel-based rendering that discards voxels which do not contribute to the final image, thereby reducing the instance count and rendering load. It operates by evaluating the scalar field value at the voxel center relative to a target isovalue to determine if the voxel lies on the visible surface. It is distinct from [Voxel Occupancy Test](1281-voxel-occupancy-test.md), which only determines if a point is within the occupied volume, by specifically filtering for surface-adjacent or surface-contained voxels based on scalar thresholds.

## How it works
**Inputs and Outputs:**
- Input: A list of occupied voxels, where each voxel has a 3D integer coordinate $(x, y, z)$ and an associated scalar field value $f$ (isovalue) at its center.
- Output: A filtered list of voxels retained for rendering.

**Data Structures:**
- `voxels`: A list or array of objects/structs containing fields `x` (int), `y` (int), `z` (int), and `isovalue` (float).

**Steps:**
1. Initialize an empty list `visible_voxels`.
2. Iterate through each voxel $v$ in the input `voxels` list.
3. Retrieve the scalar field value $f_v$ at the center of voxel $v$.
4. Apply the culling condition based on the clipping mode:
   - **Inside Clipping:** If $f_v < 0.0$, discard $v$ [S1, unofficial].
   - **Outside Clipping:** If $f_v > 0.0$, discard $v$ [S1, unofficial].
   - *Note:* The threshold $0.0$ implies the scalar field is normalized or centered around the isosurface. If a different isovalue $I$ is used, the condition becomes $f_v < I$ or $f_v > I$ respectively.
5. If the voxel is not discarded, append $v$ to `visible_voxels`.
6. Return `visible_voxels`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_visibility_culling(voxels, mode="inside"):
    """
    Cull voxels based on their scalar field value at the center.
    
    Args:
        voxels: List of dicts/objects with keys 'x', 'y', 'z', 'isovalue'.
        mode: 'inside' to keep voxels with isovalue >= 0.0, 
              'outside' to keep voxels with isovalue <= 0.0.
    
    Returns:
        List of retained voxels.
    """
    visible_voxels = []
    
    for v in voxels:
        f = v['isovalue']
        
        if mode == "inside":
            # Discard if isovalue at center is less than 0.0 [S1, unofficial]
            if f < 0.0:
                continue
        elif mode == "outside":
            # Discard if isovalue at center is greater than 0.0 [S1, unofficial]
            if f > 0.0:
                continue
        else:
            raise ValueError("Mode must be 'inside' or 'outside'")
            
        visible_voxels.append(v)
        
    return visible_voxels
```

**Parameters:**
- `mode`: String, either "inside" or "outside". Default is "inside".
- Threshold: Fixed at $0.0$ in the source implementation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point comparisons should account for epsilon if the scalar field is noisy. The source uses strict inequality ($< 0.0$, $> 0.0$) [S1, unofficial].
- **Boundary Voxels:** Voxels on the boundary of the volume may have undefined neighbors if neighbor-based culling is used (see Subtopics).
- **Degenerate Input:** Empty voxel lists return empty lists.

## Subtopics

### Interior Voxel Culling
A variant that removes voxels completely surrounded by other occupied voxels on all 6 faces (top, bottom, left, right, front, back) to reduce instance count without visual impact [S2, unofficial]. It is chosen when the rendering pipeline can handle occluded internal voxels or when the goal is to minimize draw calls for solid objects. It differs from basic scalar culling by using spatial topology rather than scalar values.
- **Definition:** A voxel is marked as fully enclosed if all 6 immediate neighbors exist in the occupied set [S2, unofficial].
- **Data Structures:**
  - `occupied`: A `HashSet<Long>` storing packed coordinate keys of all initially occupied voxels [S2, unofficial].
  - `visible`: A `List<Voxel>` storing retained voxels [S2, unofficial].
- **Steps:**
  1. Build `occupied` by packing each voxel's $(x, y, z)$ into a unique long key.
  2. For each voxel, check if all 6 neighbors $(x\pm1, y, z)$, $(x, y\pm1, z)$, $(x, y, z\pm1)$ exist in `occupied`.
  3. If all 6 neighbors are present, mark the voxel as fully enclosed and exclude it from `visible` [S2, unofficial].
- **Packing Formula:**
  $$ \text{key} = (x_{\text{offset}} \ll 42) \mid (y_{\text{offset}} \ll 21) \mid z_{\text{offset}} $$
  Where $x_{\text{offset}} = x + 1,000,000$, $y_{\text{offset}} = y + 1,000,000$, $z_{\text{offset}} = z + 1,000,000$ [S2, unofficial].
- **Pitfall:** Coordinate packing assumes coordinates fit in 20 bits each (up to ~1M per axis). Negative coordinates are handled by adding a 1,000,000 offset before packing [S2, unofficial].
- **Complexity:** $O(N)$ time for building the hash set and $O(N)$ for culling, where $N$ is the number of voxels [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for scalar-based culling, where $N$ is the number of voxels. Interior culling is also $O(N)$ but with higher constant factors due to hash set lookups.
- **Memory Complexity:** $O(N)$ for storing the voxel list. Interior culling requires an additional $O(N)$ for the hash set.
- **Strengths:** Reduces the number of primitives sent to the GPU, improving rendering performance for large volumes. Interior culling significantly reduces instance count for solid objects.
- **Limitations:** Scalar culling may remove voxels that are technically inside the volume but contribute to surface detail if the scalar field is noisy. Interior culling requires a complete set of occupied voxels and may fail for sparse or hollow structures.
- **When to use:** Use scalar culling when the scalar field accurately represents the surface. Use interior culling for solid, closed manifolds where internal voxels are not visible.

## Background
Voxel visibility culling is a common optimization in voxel-based rendering pipelines. The specific "Inside/Outside Clipping" method described here is implemented in the `vinceniko/Isosurface` project [S1, unofficial]. The "Interior Voxel Culling" variant is implemented in the `Il-castor/VoxelRenderer` project [S2, unofficial]. Both are unofficial implementations found in GitHub repositories.

## Sources
[S1] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S2] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
