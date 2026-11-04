# Point-in-Box Inclusion Test

Also called: Point Array Box Partitioning, annBoxSplit

## Overview
The Point-in-Box Inclusion Test determines which points from a given array lie within an orthogonal bounding box. It serves as the core primitive for [Voxel Occupancy Test](1281-voxel-occupancy-test.md), filtering points to identify voxel occupancy. Unlike [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) which uses distance metrics, this method relies on strict axis-aligned boundary checks. It differs from [Sparse AABB Collision Test](3047-sparse-aabb-collision-test.md) by operating on a static point array rather than dynamic collision shapes.

## How it works
**Inputs:**
- `PP`: An array of $n$ points, where each point has 3 coordinates $(x, y, z)$.
- `box`: An orthogonal bounding box defined by minimum bounds `min` and maximum bounds `max` for each dimension $d \in \{x, y, z\}$.

**Outputs:**
- `n_in`: The count of points inside or on the boundary of the box.
- The input array `PP` is partitioned such that the first `n_in` points are inside, and the remaining are outside.

**Data Structures:**
- `PP`: Array of points.
- `box`: Struct with fields `min` and `max`, each being a 3D vector.

**Algorithm:**
1. Initialize two indices: `l = 0` (left) and `r = n - 1` (right).
2. Loop while `l < n` and `r >= 0`:
   a. Increment `l` while `l < n` and point `PP[l]` is inside the box on all dimensions.
   b. Decrement `r` while `r >= 0` and point `PP[r]` is NOT inside the box on all dimensions.
   c. If `l > r`, break the loop.
   d. Swap `PP[l]` and `PP[r]`.
   e. Increment `l` and decrement `r`.
3. Set `n_in = l`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_in_box_inclusion_test(PP, box):
    n = len(PP)
    l = 0
    r = n - 1
    
    while l < n and r >= 0:
        # Move l forward while points are inside
        while l < n and box_inside(box, PP[l]):
            l += 1
        
        # Move r backward while points are outside
        while r >= 0 and not box_inside(box, PP[r]):
            r -= 1
            
        # If pointers crossed, we are done
        if l > r:
            break
            
        # Swap the outside point at l with the outside point at r
        # (Actually, l is outside, r is outside? No: l stopped at outside, r stopped at inside? 
        # Let's re-read S1 carefully: "while l < n and box.inside(dim, PP(l)), l++" -> l stops at first OUTSIDE.
        # "while r >= 0 and !box.inside(dim, PP(r)), r--" -> r stops at first INSIDE.
        # So we swap an OUTSIDE (at l) with an INSIDE (at r).
        
        swap(PP, l, r)
        l += 1
        r -= 1
        
    return l
```

**Helper `box_inside`:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def box_inside(box, point):
    for dim in ['x', 'y', 'z']:
        if point[dim] < box.min[dim] or point[dim] > box.max[dim]:
            return False
    return True
```

**Edge Cases:**
- Points exactly on the boundary are considered inside (inclusive check).
- Empty point array ($n=0$) returns 0.

## Subtopics

### Point-in-Box Inclusion Test with Dimension Selection
Finds the spatial dimension with the largest spread among all dimensions in a point set before partitioning. This heuristic aims to optimize the partitioning order or selection for subsequent operations by identifying the axis of maximum variance. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** $O(n)$ in the worst case, as each point is visited at most once by the two pointers.
- **Memory Complexity:** $O(1)$ auxiliary space if performed in-place.
- **Strengths:** Simple, fast, and cache-friendly due to linear scan.
- **Limitations:** Does not sort points, only partitions. Performance degrades if the box is very small or very large relative to the point distribution, though the linear scan remains constant time.

## Background
Originated in the context of 3D reconstruction and dual contouring implementations, specifically within the `DualContouring-3DRendring` project. It is used to efficiently filter surfels or points against voxel boundaries during mesh generation. [S1, unofficial]

## Sources
[S1] bouguerra-khalil/DualContouring-3DRendring, https://github.com/bouguerra-khalil/DualContouring-3DRendring, fetched 2026-10-26 (unofficial)
[S2] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
