# Point-in-Box Inclusion Test

Also called: Point Array Box Partitioning, annBoxSplit

## Overview
The Point-in-Box Inclusion Test partitions a point array about an orthogonal bounding box, returning the count of points inside or on the boundary [S1, unofficial]. It is used within [Voxel Occupancy Test](1281-voxel-occupancy-test.md) to filter points for voxel occupancy determination [S1, unofficial].

## How it works
**Inputs:**
- `PP`: An array of $n$ points, where each point has 3 coordinates $(x, y, z)$ [S1, unofficial].
- `box`: An orthogonal bounding box defined by minimum bounds `min` and maximum bounds `max` for each dimension $d \in \{x, y, z\}$ [S1, unofficial].

**Outputs:**
- `n_in`: The count of points inside or on the boundary of the box [S1, unofficial].
- The input array `PP` is partitioned such that the first `n_in` points are inside, and the remaining are outside [S1, unofficial].

**Data Structures:**
- `PP`: Array of points [S1, unofficial].
- `box`: Struct with fields `min` and `max`, each being a 3D vector [S1, unofficial].

**Algorithm:**
1. Initialize two indices: `l = 0` (left) and `r = n - 1` (right) [S1, unofficial].
2. Loop while `l < n` and `r >= 0`:
   a. Increment `l` while `l < n` and point `PP[l]` is inside the box on all dimensions [S1, unofficial].
   b. Decrement `r` while `r >= 0` and point `PP[r]` is NOT inside the box on all dimensions [S1, unofficial].
   c. If `l > r`, break the loop [S1, unofficial].
   d. Swap `PP[l]` and `PP[r]` [S1, unofficial].
   e. Increment `l` and decrement `r` [S1, unofficial].
3. Set `n_in = l` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_in_box_inclusion_test(PP, box):
    n = len(PP)
    l = 0
    r = n - 1
    
    while l < n and r >= 0:
        while l < n and box_inside(box, PP[l]):
            l += 1
        while r >= 0 and not box_inside(box, PP[r]):
            r -= 1
        if l > r:
            break
        PP[l], PP[r] = PP[r], PP[l]
        l += 1
        r -= 1
    return l

def box_inside(box, point):
    for dim in ['x', 'y', 'z']:
        if point[dim] < box.min[dim] or point[dim] > box.max[dim]:
            return False
    return True
```

**Edge Cases:**
- Points exactly on the boundary are considered inside (inclusive check) [S1, unofficial].
- Empty point array ($n=0$) returns 0 [G].

## Subtopics

### Point-in-Box Inclusion Test with Dimension Selection
Finds the spatial dimension with the largest spread among all dimensions in a point set before partitioning [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Originated in the context of 3D reconstruction and dual contouring implementations, specifically within the `DualContouring-3DRendring` project [S1, unofficial].

## Sources
[S1] bouguerra-khalil/DualContouring-3DRendring, https://github.com/bouguerra-khalil/DualContouring-3DRendring, fetched 2026-10-26 (unofficial)
[S2] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
