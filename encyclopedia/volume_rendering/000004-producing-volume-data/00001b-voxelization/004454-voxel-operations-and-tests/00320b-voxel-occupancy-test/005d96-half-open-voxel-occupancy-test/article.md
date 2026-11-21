# Half-open Voxel Occupancy Test

Also called: Half-open depth interval for voxel selection, Rule F half-open interval, integersWithProductInHalfOpenInterval

## Overview
The Half-open Voxel Occupancy Test is a voxel selection method that uses a half-open interval $[min, max)$ along the depth normal to ensure each voxel belongs to exactly one of two consecutive fills, preventing gaps or overlaps at boundaries [S1, unofficial]. Unlike the open interval which varies layer count by plane position (giving one layer more or less), the half-open interval of width $F$ holds exactly $F / T_v$ layers regardless of the plane's position between two layers [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: A set of candidate voxels (or a slab definition) and a depth interval type ('open' or 'half-open').
- Output: A boolean mask indicating which voxels are selected by the fill operation.

**Data Structures**
The implementation relies on a function `integersWithProductInHalfOpenInterval` located in `indexSpaceSlab.ts` [S1, unofficial]. This function operates alongside an existing open-interval variant. The calling functions `buildIndexSpaceSlab` and `iterateVoxelsInShape` accept a `depthInterval` option [S1, unofficial].

**Steps**
1.  Determine the depth interval type. For Rule F (brush fill), the type is explicitly set to 'half-open' [S1, unofficial]. Other rules (e.g., Rule M, Rule D) may retain the 'open' default [S1, unofficial].
2.  Calculate the depth bounds for the voxel slab. The interval width is maintained at exactly $2 \times halfWidth$ [S1, unofficial].
3.  Apply the epsilon shift to the interval boundaries rather than narrowing the interval width [S1, unofficial]. This ensures the geometric width remains constant while shifting the selection window [S1, unofficial].
4.  Test each voxel against the half-open interval $[min, max)$ [S1, unofficial].
5.  Select the voxel if its depth coordinate $d$ satisfies $min \le d < max$ [S1, unofficial].

**Formula**
The selection condition for a voxel at depth $d$ is:
$$ min \le d < max $$
where the interval width is:
$$ max - min = 2 \times halfWidth $$
The epsilon parameter shifts $min$ and $max$ equally, preserving the width [S1, unofficial].

**Parameters**
- `depthInterval`: Enum value 'open' (default) or 'half-open' [S1, unofficial].
- `halfWidth`: Defines the slab thickness. The total width is $2 \times halfWidth$ [S1, unofficial].

**Edge Cases and Pitfalls**
- **Gaps at Oblique Angles**: Using an open interval causes gaps in voxel coverage at oblique angles. For a 25x25x25 volume with 1mm voxels at 45 degrees oblique, 7800 of 15625 voxels were never written by any fill using an open interval, whereas the half-open interval results in 0 unwritten voxels [S1, unofficial].
- **Layer Count Consistency**: The open interval causes the number of layers to vary by plane position. The half-open interval ensures a consistent count of $F / T_v$ layers [S1, unofficial].
- **Rule M Exception**: Rule M requires the open interval because its widened slab must report both layers for a plane midway between two layers [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_voxels_half_open(voxels, min_depth, max_depth):
    """
    Select voxels where depth is in [min_depth, max_depth).
    Args:
        voxels: List of voxel objects with a 'depth' attribute.
        min_depth: Lower bound (inclusive).
        max_depth: Upper bound (exclusive).
    Returns:
        List of selected voxels.
    """
    selected = []
    for voxel in voxels:
        if min_depth <= voxel.depth < max_depth:
            selected.append(voxel)
    return selected

def build_index_space_slab(depth_interval='open'):
    """
    Build the index space slab with the specified depth interval.
    Args:
        depth_interval: 'open' or 'half-open'.
    Returns:
        Slab configuration.
    """
    if depth_interval == 'half-open':
        # Use integersWithProductInHalfOpenInterval logic
        # Interval width is exactly 2 * halfWidth
        # Epsilon shifts boundaries, does not narrow width
        return create_slab_half_open()
    else:
        # Use existing open interval logic
        return create_slab_open()
```

## Subtopics

### [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md)
Tests if points are within a box. Simpler than half-open test as it does not handle depth intervals for fills.

### [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md)
Generates kernel for circular shapes. Does not address depth interval gaps.

### [Voxel Filled Check](1705-voxel-filled-check.md)
Checks if a voxel is filled. Does not handle the boundary selection logic for consecutive fills.

### [Sparse AABB Collision Test](3047-sparse-aabb-collision-test.md)
Tests collision in sparse structures. Different scope than voxel occupancy for rendering fills.

### [Solidity Check](4229-solidity-check.md)
Checks if a shape is solid. Does not address the specific depth interval gap issue.

### [Sphere-Clamped Voxel Neighbor Search](5131-sphere-clamped-voxel-neighbor-search.md)
Searches neighbors within a sphere. Not related to depth interval selection.

### [Voxel Collision Test](5774-voxel-collision-test.md)
Tests for collisions. Different application than fill coverage.

### [Mask Cache Lookup](6434-mask-cache-lookup.md)
Uses a cache for mask lookups. Does not define the interval logic itself.

### [Voxel Visibility Culling](8235-voxel-visibility-culling.md)
Culls invisible voxels. Complementary but distinct from the occupancy test logic.

## Operating characteristics
- **Time Complexity**: Linear in the number of candidate voxels, $O(N)$, where $N$ is the number of voxels in the slab [S1, unofficial].
- **Memory Complexity**: $O(N)$ to store the selection mask or selected voxel list [S1, unofficial].
- **Measured Performance**: In a 25x25x25 volume with 1mm voxels at 45 degrees oblique, the half-open interval resulted in 0 unwritten voxels, compared to 7800 unwritten voxels with the open interval [S1, unofficial].
- **Strengths**: Eliminates gaps in voxel coverage at oblique angles; ensures consistent layer count regardless of plane position [S1, unofficial].
- **Limitations**: Not suitable for rules (like Rule M) that require reporting both layers for a plane midway between two layers [S1, unofficial].
- **When to Use**: Use for Rule F (brush fill) and other cases where continuous coverage without gaps is required [S1, unofficial].
- **When Not to Use**: Do not use for Rule M or Rule D, which require the open interval [S1, unofficial].

## Background
The method was introduced in PR #2743, commit 7a5b0fba3, by wayfarer3130 on Sep 14, 2026, as part of the cornerstone3D project [S1, unofficial]. It was implemented to correct gaps in voxel coverage observed during oblique slice rendering. The change specifically affects Rule F, while Rule M and Rule D retain the open interval [S1, unofficial].

## Sources
[S1] A volume viewport steps an oblique slice by the L2 spacing, and shows the same voxels on two slices, https://github.com/cornerstonejs/cornerstone3D/issues/2912, fetched 2026-10-27 (unofficial)
