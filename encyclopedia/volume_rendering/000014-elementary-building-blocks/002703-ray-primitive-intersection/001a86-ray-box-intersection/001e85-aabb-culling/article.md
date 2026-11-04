# AABB Culling

Also called: 8-point frustum test, Box culling

## Overview
AABB Culling determines if a box defined by 8 vertices is completely outside a viewing box along any axis, returning false if culled and true otherwise [S1, unofficial]. It serves as a fast rejection test in volume rendering and ray tracing to avoid expensive intersection calculations for primitives that are clearly outside the view frustum or a specific spatial region. Unlike [Ray-box intersection](776-ray-box-intersection.md), which computes precise entry and exit distances for potential hits, this method performs a coarse binary check to discard objects early.

## How it works
**Inputs:**
- `points`: An array of 8 vectors `[x, y, z]` representing the vertices of the object box [S1, unofficial].
- `box`: A viewing box defined by 6 scalar values `[min_x, max_y, min_y, max_x, min_z, max_z]` (indices 0-5) and potentially additional bounds at indices 3-5 depending on implementation specifics [S1, unofficial]. Note: The source notation `box[3]`, `box[0]`, etc., implies a specific packing order where `box[0]` is min-x, `box[3]` is max-x, `box[1]` is min-y, `box[4]` is max-y, `box[2]` is min-z, and `box[5]` is max-z.

**Outputs:**
- Boolean: `true` if the box intersects or overlaps the viewing box; `false` if it is completely outside.

**Data Structures:**
- `points`: Array of 8 elements, each a 3-component vector.
- `box`: Array of 6 or more scalar values defining the axis-aligned viewing volume boundaries.

**Steps:**
1. Initialize a counter `out = 0`.
2. Loop `i` from 0 to 7 (for each of the 8 vertices):
   a. If `points[i][0] > box[3]`, increment `out`.
   b. If `out == 8`, return `false` (all points are outside the max-x bound).
3. Repeat the check for other axes:
   a. If `points[i][0] < box[0]`, increment `out`.
   b. If `points[i][1] > box[4]`, increment `out`.
   c. If `points[i][1] < box[1]`, increment `out`.
   d. If `points[i][2] > box[5]`, increment `out`.
   e. If `points[i][2] < box[2]`, increment `out`.
   *(Note: The source description implies a sequential check where `out` is incremented for each vertex that fails a specific bound check, and if all 8 vertices fail the same bound check, the object is culled. However, the pseudocode below clarifies the logic as checking if ALL points are outside a specific half-space.)*

**Corrected Logic Interpretation:**
The source describes a test where `out` counts how many points are outside a specific boundary. If all 8 points are outside the same boundary (e.g., all `x > max_x`), the box is culled. The algorithm checks each of the 6 planes of the viewing box. If the box is completely outside any one plane, it returns `false`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aabb_culling(points, box):
    """
    points: list of 8 tuples (x, y, z)
    box: list of 6 scalars [min_x, max_y, min_y, max_x, min_z, max_z]
         Note: Source indices are [0]=min_x, [3]=max_x, [1]=min_y, [4]=max_y, [2]=min_z, [5]=max_z
    """
    # Extract bounds for clarity
    min_x = box[0]
    max_x = box[3]
    min_y = box[1]
    max_y = box[4]
    min_z = box[2]
    max_z = box[5]

    # Check if all points are outside the viewing box along any axis
    # X-axis: All points > max_x
    all_above_max_x = all(p[0] > max_x for p in points)
    if all_above_max_x:
        return False

    # X-axis: All points < min_x
    all_below_min_x = all(p[0] < min_x for p in points)
    if all_below_min_x:
        return False

    # Y-axis: All points > max_y
    all_above_max_y = all(p[1] > max_y for p in points)
    if all_above_max_y:
        return False

    # Y-axis: All points < min_y
    all_below_min_y = all(p[1] < min_y for p in points)
    if all_below_min_y:
        return False

    # Z-axis: All points > max_z
    all_above_max_z = all(p[2] > max_z for p in points)
    if all_above_max_z:
        return False

    # Z-axis: All points < min_z
    all_below_min_z = all(p[2] < min_z for p in points)
    if all_below_min_z:
        return False

    # If not completely outside any single plane, it intersects or overlaps
    return True
```

**Parameters:**
- `points`: Array of 8 vectors [x, y, z] representing the vertices of the object box [S1, unofficial].

**Edge Cases and Pitfalls:**
- The method assumes the `points` array contains the 8 corners of the AABB. If the box is not axis-aligned, this simple component-wise test is invalid without transformation [S1, unofficial].
- Precision issues may arise if points lie exactly on the boundary; the strict inequality (`>`, `<`) may need adjustment to `>=` or `<=` depending on whether touching is considered an intersection or cull [G].

## Subtopics

### [Bounding Box Culling](1806-bounding-box-culling.md)
Bounding Box Culling is a view-dependent pre-processing step that filters out 3D Gaussians whose projected 2D footprint does not intersect the camera viewport. It differs from general [AABB Culling](1789-aabb-culling.md) by operating on the projected 2D ellipse of a Gaussian rather than a static 3D axis-aligned box, thereby enabling early rejection of primitives that are off-screen in screen space.

### Subspace Culling for Ray-Box Intersection
Subspace Culling reduces false-positive hits in ray-box intersection tests by embedding a binary voxel data structure into the volume, allowing cheap ray-voxel intersection via bitwise AND operations [S2, unofficial]. It represents the ray as a conservative voxel volume and performs intersection checks using bitwise AND between the ray's voxel volume and the embedded voxel data structure [S2, unofficial]. This method is applicable to hierarchical data structures such as Bounding Volume Hierarchies (BVH) to reduce the number of intersections during traversal [S2, unofficial]. It compresses the voxel data using a lookup table to achieve significant false-positive reductions with a small memory footprint [S2, unofficial]. Unlike oriented bounding boxes which reduce false positives but are computationally expensive, this method uses cheap bitwise operations on voxel data [S2, unofficial]. A pitfall of standard AABBs is that they produce extra empty space, leading to false-positive hits for thin and diagonal primitives, which this method aims to reduce [S2, unofficial].

## Operating characteristics
- **Time Complexity:** O(1) per box, as it involves a fixed number of comparisons (8 vertices × 6 planes).
- **Memory Complexity:** O(1) auxiliary space.
- **Strengths:** Extremely fast rejection test; minimal computational overhead.
- **Limitations:** Coarse approximation; does not provide intersection distances; assumes axis-aligned boxes.
- **When to use:** As a pre-filter before expensive ray-box intersection tests [S1, unofficial].
- **When not to use:** When precise intersection data is required or when boxes are not axis-aligned [S1, unofficial].

## Background
The method originates from practical implementations in WebGL-based volume rendering and marching cubes algorithms, specifically noted in the `webgl-marching-cubes` repository [S1, unofficial]. It is a standard technique in computer graphics for frustum culling of axis-aligned bounding boxes.

## Sources
[S1] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S2] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
