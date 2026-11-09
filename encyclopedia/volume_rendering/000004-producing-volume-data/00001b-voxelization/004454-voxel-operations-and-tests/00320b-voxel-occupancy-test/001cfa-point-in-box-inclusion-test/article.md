# Point-in-Box Inclusion Test

Also called: Point Array Box Partitioning, annBoxSplit

## Overview
The Point-in-Box Inclusion Test partitions a set of points by determining which lie within an orthogonal bounding box. It serves as the core primitive for voxel occupancy tests, returning the count of points inside or on the boundary of the box. Unlike sibling methods that may use spatial acceleration structures or kernel approximations, this method performs a direct geometric containment check, often optimized via in-place partitioning to support subsequent sorting or hierarchical decomposition.

## How it works
**Inputs:**
- `PP`: An array of $N$ points, where each point $P_i$ has coordinates $(x_i, y_i, z_i)$.
- `box`: An axis-aligned bounding box defined by minimum bounds $\mathbf{min} = (min_x, min_y, min_z)$ and maximum bounds $\mathbf{max} = (max_x, max_y, max_z)$.

**Outputs:**
- `n_in`: The number of points inside or on the boundary of the box.
- The input array `PP` is typically reordered such that the first `n_in` points are those inside the box, and the remaining points are outside.

**Data Structures:**
- The point array `PP` is stored as a contiguous block of memory. Each point is a struct or tuple of three floating-point values.

**Algorithm Steps:**
The method uses a two-pointer approach to partition the array in-place.

1. Initialize two indices: `l = 0` and `r = n - 1`.
2. Loop while `l < n` and `r >= 0`:
   a. Increment `l` while `l < n` and point `PP[l]` is inside the box along the current dimension being tested (or all dimensions, depending on implementation context; the source implies a dimension-wise or full-box check). Specifically: `while l < n and box.inside(dim, PP(l)): l++`.
   b. Decrement `r` while `r >= 0` and point `PP[r]` is NOT inside the box: `while r >= 0 and !box.inside(dim, PP(r)): r--`.
   c. If `l > r`, break the loop.
   d. Swap `PP[l]` and `PP[r]`.
   e. Increment `l` and decrement `r`.
3. Set `n_in = l`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_in_box_inclusion_test(PP, box):
    """
    PP: List of points, each point is a tuple/list of 3 floats (x, y, z).
    box: Object with method inside(dim, point) or attributes min/max.
         For full 3D box check, inside checks all axes.
    Returns: (n_in, PP) where PP is partitioned.
    """
    n = len(PP)
    l = 0
    r = n - 1
    
    while l < n and r >= 0:
        # Move l forward while point is inside the box
        while l < n and is_inside_box(PP[l], box):
            l += 1
        
        # Move r backward while point is outside the box
        while r >= 0 and not is_inside_box(PP[r], box):
            r -= 1
            
        # If pointers crossed, we are done
        if l > r:
            break
            
        # Swap the outside point at l with the inside point at r
        PP[l], PP[r] = PP[r], PP[l]
        l += 1
        r -= 1
        
    n_in = l
    return n_in, PP

def is_inside_box(point, box):
    """
    Checks if a point is inside or on the boundary of the box.
    Assumes box has min and max attributes for x, y, z.
    """
    return (box.min.x <= point.x <= box.max.x and
            box.min.y <= point.y <= box.max.y and
            box.min.z <= point.z <= box.max.z)
```

**Parameters:**
- `box`: The bounding volume. Typical values depend on the voxel grid resolution and scene bounds.
- `dim`: In some variants, a specific dimension is tested. If not specified, all three dimensions are tested.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point comparisons for boundary inclusion (`<=`, `>=`) must be consistent to avoid points being counted or missed due to rounding errors.
- **Empty Array:** If $N=0$, the loop does not execute, and `n_in` is 0.
- **All Inside/Outside:** If all points are inside, `l` increments to `n`, `r` stays at `n-1` (or decrements if not careful, but the logic `l > r` breaks it). If all are outside, `l` stays 0, `r` decrements to -1.

## Subtopics

### [Point-in-Box Inclusion Test via Sorted Indices and Bisect](7081-point-in-box-inclusion-test-via-sorted-indices-and-bisect.md)
This method accelerates voxel occupancy tests by pre-sorting point cloud coordinates along each axis, enabling $O(\log N)$ range queries via binary search instead of $O(N)$ linear scans. It differs from the parent [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md) by optimizing the inner loop for repeated queries on static data, trading $O(N)$ preprocessing time for faster per-query performance.

### Point-in-Box Inclusion Test with Dimension Selection
This variant finds the spatial dimension with the largest spread among all dimensions in a point set. This selection is used to prioritize partitioning along the axis with the greatest variance, potentially improving cache locality or balance in subsequent hierarchical structures.

## Operating characteristics
- **Time Complexity:** $O(N)$ per query for the basic partitioning method, where $N$ is the number of points. The variant with sorted indices requires $O(N \log N)$ preprocessing for sorting, but $O(\log N)$ per query.
- **Memory Complexity:** $O(1)$ auxiliary space for the in-place partitioning method. The sorted variant requires $O(N)$ space for indices.
- **Strengths:** Simple implementation, low memory overhead for the basic method, effective for one-time occupancy checks.
- **Limitations:** Linear scan is slow for repeated queries on large static datasets. The basic method does not handle dynamic point sets efficiently without resorting to more complex structures.
- **When to use:** Use the basic method for single-pass voxelization or when memory is constrained. Use the sorted variant for scenes with static geometry and frequent occupancy queries.

## Background
The method originates from computational geometry techniques for point set partitioning. The specific implementation details, including the two-pointer partitioning logic, are documented in the context of 3D rendering and dual contouring pipelines.

## Sources
[S1] bouguerra-khalil/DualContouring-3DRendring, https://github.com/bouguerra-khalil/DualContouring-3DRendring, fetched 2026-10-26 (unofficial)
[S2] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
