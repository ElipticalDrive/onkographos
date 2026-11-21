# Ray-AABB Intersection and Grid Traversal Setup

Also called: traverse_grids initialization

## Overview
This method prepares rays for volume rendering by determining which spatial grids (or voxels) each ray intersects and calculating the entry and exit distances for traversal. It builds on [Slab-based AABB Intersection](19515-slab-based-aabb-intersection.md) to compute precise intersection intervals. Unlike [Ray-Cube Intersection](2466-ray-cube-intersection.md) which assumes uniform cubic cells, or [AABB Ray Intersection](11086-aabb-ray-intersection.md) which may provide generic bounding checks, this setup specifically handles multiple grid structures and prepares sorted traversal intervals for efficient ray marching.

## How it works
**Inputs:**
- A set of rays, each defined by an origin $\mathbf{o}$ and direction $\mathbf{d}$.
- A list of axis-aligned bounding boxes (AABBs) representing the grids or volume partitions.
- Near and far clipping plane distances ($t_{near}, t_{far}$).
- A `miss_value` constant for rays that do not intersect an AABB.

**Outputs:**
- For each ray, a list of intersection intervals $(t_{min}, t_{max})$ for each intersected AABB.
- A hit flag indicating whether the ray intersects any AABB.

**Data Structures:**
- `t_mins`: Array of size $N_{rays} \times N_{grids}$ storing entry distances.
- `t_maxs`: Array of size $N_{rays} \times N_{grids}$ storing exit distances.
- `hits`: Boolean array of size $N_{rays} \times N_{grids}$ indicating intersection validity.

**Steps:**
1. **Compute Intersections:** For each ray $i$ and each grid $j$, compute the intersection distances $t_{min}^{i,j}$ and $t_{max}^{i,j}$ using slab-based AABB intersection [S1, unofficial].
2. **Clamp to View Frustum:** Clamp $t_{min}^{i,j}$ and $t_{max}^{i,j}$ to the range $[t_{near}, t_{far}]$. If $t_{min}^{i,j} > t_{far}$ or $t_{max}^{i,j} < t_{near}$, the ray does not traverse the grid within the visible range.
3. **Handle Misses:** If the ray does not intersect the AABB (hit flag is false), set $t_{min}^{i,j} = t_{max}^{i,j} = \text{miss\_value}$ [S1, unofficial]. The `miss_value` is typically a large positive number (e.g., infinity) to ensure these intervals are sorted to the end of the traversal list.
4. **Sort Intervals:** For each ray, concatenate all $(t_{min}^{i,j}, t_{max}^{i,j})$ pairs from all grids. Sort these intervals by $t_{min}$ to prepare for sequential traversal. This ensures that the ray processes grids in the order it encounters them.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_grid_traversal(rays, grids, t_near, t_far, miss_value=float('inf')):
    """
    rays: List of tuples (origin, direction)
    grids: List of AABBs (min_corner, max_corner)
    t_near: Near plane distance
    t_far: Far plane distance
    miss_value: Value for non-intersecting rays
    Returns: List of sorted intervals for each ray
    """
    all_intervals = []
    
    for ray in rays:
        ray_intervals = []
        for grid in grids:
            # Compute intersection using slab method [S1, unofficial]
            t_min, t_max, hit = slab_aabb_intersect(ray.origin, ray.direction, grid.min, grid.max)
            
            if hit:
                # Clamp to view frustum
                t_min = max(t_min, t_near)
                t_max = min(t_max, t_far)
                
                # If the interval is valid after clamping, add it
                if t_min < t_max:
                    ray_intervals.append((t_min, t_max))
            else:
                # No intersection: use miss_value to sort to end
                ray_intervals.append((miss_value, miss_value))
        
        # Sort intervals by entry distance
        ray_intervals.sort(key=lambda x: x[0])
        all_intervals.append(ray_intervals)
        
    return all_intervals
```

**Parameters:**
- `miss_value`: Typically infinity or a value larger than any possible $t_{far}$ [S1, unofficial].

**Edge Cases:**
- **Degenerate AABBs:** If an AABB has zero volume, the intersection logic must handle it gracefully (usually resulting in no intersection or a single point).
- **Ray Parallel to Slab:** If the ray direction is parallel to a slab face, the intersection calculation must handle division by zero or near-zero denominators.

## Subtopics

### Ray Truncation to Volume Bounds
Truncates a ray defined by source and detector points to the volume's AABB, returning a line segment with voxel-space origin and step count [S2, unofficial].
- **Steps:**
  1. Call `aabb_intersection` with `ray.source`, `ray.detector`, `v.physical_lengths()`, and `vec<D, T>(v.origin())` [S2, unofficial].
  2. If no intersection is found, return an empty optional [S2, unofficial].
  3. Convert `ray.source`, `ray.detector`, and intersection origin to voxel coordinates using `to_voxel` [S2, unofficial].
  4. Calculate the number of steps as the integer distance from `voxel_source` to `voxel_origin` plus one [S2, unofficial].
- **Formula:** $steps = \lfloor \|v_{source} - v_{origin}\| \rfloor + 1$ [S2, unofficial].
- **Pitfall:** The step calculation truncates the floating-point distance to an integer before adding one, which may slightly underestimate or overestimate the exact number of voxels traversed depending on rounding [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \times N_{grids})$ for intersection computation, plus $O(N_{rays} \times N_{grids} \log N_{grids})$ for sorting intervals per ray.
- **Memory Complexity:** $O(N_{rays} \times N_{grids})$ to store intervals.
- **Strengths:** Efficiently handles multiple grid structures; prepares data for fast sequential traversal.
- **Limitations:** Sorting overhead can be significant for large numbers of grids; miss_value handling requires careful choice to avoid overflow or incorrect sorting.
- **When to Use:** When rendering volumes with multiple distinct grid structures or partitions.
- **When Not to Use:** For simple single-grid volumes where direct traversal is sufficient.

## Background
This method is commonly used in volume rendering pipelines that involve spatial partitioning or multi-resolution grids. It ensures that rays are processed in the correct order through the volume, which is critical for correct accumulation of color and opacity.

## Sources
[S1] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
[S2] jwbuurlage/TPT, https://github.com/jwbuurlage/TPT, fetched 2026-10-26 (unofficial)
