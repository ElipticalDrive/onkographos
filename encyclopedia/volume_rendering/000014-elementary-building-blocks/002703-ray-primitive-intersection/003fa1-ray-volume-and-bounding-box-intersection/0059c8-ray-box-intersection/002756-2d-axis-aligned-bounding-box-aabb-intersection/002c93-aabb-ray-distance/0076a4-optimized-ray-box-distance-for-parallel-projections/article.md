# Optimized Ray-Box Distance for Parallel Projections

Also called: Distance Computation for Entry and Exit Points, Parallel Projection Ray-Box Distance

## Overview
This method optimizes the calculation of the Euclidean distance between ray entry and exit points within an axis-aligned bounding box (AABB) for parallel projection volume rendering [S1, unofficial]. It avoids the computationally expensive square root operation inherent in the standard distance formula by exploiting the geometric constraints of parallel rays and octree structures [S1, unofficial]. It differs from general ray-box intersection methods by precomputing direction cosines and using case-specific arithmetic shortcuts based on the octree level [S1, unofficial].

## How it works
**Inputs:**
- Ray direction vector $\vec{r} = (r_x, r_y, r_z)$.
- Bounding box defined by minimum corner $min = (min_x, min_y, min_z)$ and maximum corner $max = (max_x, max_y, max_z)$.
- Entry point $a = (a_x, a_y, a_z)$ and exit point $b = (b_x, b_y, b_z)$ on the box surface.
- Octree level $i$ of the node (where $i=0$ is the root).
- $d_0$: Intersection length of the ray with the entire dataset's bounding box (precomputed).

**Outputs:**
- Distance $d$ along the ray from $a$ to $b$.

**Data Structures:**
- Direction cosines: $\cos \alpha, \cos \beta, \cos \gamma$. These are derived from the normalized ray direction.
- Coordinates of entry $a$ and exit $b$.

**Steps:**
1. **Compute Direction Cosines:**
   Calculate the magnitude of the ray direction: $|\vec{r}| = \sqrt{r_x^2 + r_y^2 + r_z^2}$.
   Compute cosines:
   $$ \cos \alpha = \frac{r_x}{|\vec{r}|}, \quad \cos \beta = \frac{r_y}{|\vec{r}|}, \quad \cos \gamma = \frac{r_z}{|\vec{r}|} $$
   [G] Ensure $|\vec{r}|$ is not zero to avoid division by zero.

2. **Detect Intersection Case:**
   Determine if the ray passes through opposite faces or adjacent faces sharing an edge.
   - **Case 1 (Opposite Faces):** The entry point $a$ has at least two coordinates equal to the corresponding coordinates of the box's minimum corner ($min$), AND the exit point $b$ has at least two coordinates equal to the corresponding coordinates of the box's maximum corner ($max$).
   - **Case 2 (Adjacent Faces):** The ray intersects two faces sharing a common edge. This is the default if Case 1 is false.

3. **Calculate Distance:**
   - **If Case 1:** Use the octree level formula:
     $$ d = \frac{d_0}{i + 1} $$
     [S1, unofficial] This relies on the property that intersection length is inversely proportional to the node level in a uniform octree.
   - **If Case 2:** Determine which axis the shared edge is parallel to. The shared edge is parallel to the axis where the coordinate difference between $a$ and $b$ is zero (or negligible due to precision).
     - If the edge is parallel to the X-axis (i.e., $a_y = b_y$ and $a_z = b_z$):
       $$ d = (b_x - a_x) \cos \alpha $$
     - If the edge is parallel to the Y-axis (i.e., $a_x = b_x$ and $a_z = b_z$):
       $$ d = (b_y - a_y) \cos \beta $$
     - If the edge is parallel to the Z-axis (i.e., $a_x = b_x$ and $a_y = b_y$):
       $$ d = (b_z - a_z) \cos \gamma $$
     [S1, unofficial] Note: The source formulas (10-12) use the coordinate difference multiplied by the cosine of the angle with the respective axis.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimized_ray_box_distance(a, b, min_corner, max_corner, i, d0, ray_dir):
    """
    a: tuple (ax, ay, az) entry point
    b: tuple (bx, by, bz) exit point
    min_corner: tuple (min_x, min_y, min_z)
    max_corner: tuple (max_x, max_y, max_z)
    i: int, octree level
    d0: float, root box intersection length
    ray_dir: tuple (rx, ry, rz)
    """
    # Step 1: Compute direction cosines
    mag = (ray_dir[0]**2 + ray_dir[1]**2 + ray_dir[2]**2)**0.5
    if mag == 0:
        return 0.0 # Guard against zero-length ray
    cos_alpha = ray_dir[0] / mag
    cos_beta = ray_dir[1] / mag
    cos_gamma = ray_dir[2] / mag

    # Step 2: Detect Case 1 (Opposite Faces)
    # Check if a matches min_corner in at least 2 dimensions
    a_match_min = sum(1 for k in range(3) if abs(a[k] - min_corner[k]) < 1e-9)
    # Check if b matches max_corner in at least 2 dimensions
    b_match_max = sum(1 for k in range(3) if abs(b[k] - max_corner[k]) < 1e-9)

    if a_match_min >= 2 and b_match_max >= 2:
        # Case 1: Opposite faces
        return d0 / (i + 1)
    else:
        # Case 2: Adjacent faces sharing an edge
        # Determine which axis the edge is parallel to
        # The edge is parallel to the axis where the coordinate does not change
        # i.e., a[k] == b[k]
        
        # Check X-axis parallel (edge along X, so Y and Z are constant)
        if abs(a[1] - b[1]) < 1e-9 and abs(a[2] - b[2]) < 1e-9:
            return (b[0] - a[0]) * cos_alpha
        # Check Y-axis parallel (edge along Y, so X and Z are constant)
        elif abs(a[0] - b[0]) < 1e-9 and abs(a[2] - b[2]) < 1e-9:
            return (b[1] - a[1]) * cos_beta
        # Check Z-axis parallel (edge along Z, so X and Y are constant)
        elif abs(a[0] - b[0]) < 1e-9 and abs(a[1] - b[1]) < 1e-9:
            return (b[2] - a[2]) * cos_gamma
        else:
            # Fallback or error: Should not happen for valid box intersections
            # Compute standard Euclidean distance as fallback [G]
            return ((b[0]-a[0])**2 + (b[1]-a[1])**2 + (b[2]-a[2])**2)**0.5
```

**Parameters:**
- $d_0$: Precomputed intersection length with the root bounding box.
- $i$: Octree level index, starting at 0 for the root.
- $\cos \alpha, \cos \beta, \cos \gamma$: Direction cosines of the parallel ray.

**Edge Cases:**
- **Zero Ray Length:** If the ray direction is zero, the distance is undefined. A guard is required [G].
- **Precision:** Floating-point comparisons for coordinate equality (e.g., $a_y = b_y$) should use a small epsilon tolerance to handle numerical inaccuracies [G].
- **Degenerate Boxes:** If the box has zero volume, the distance is zero.

## Operating characteristics
- **Time Complexity:** $O(1)$ per ray-box intersection. It replaces a square root and three subtractions/squarings with simple arithmetic operations and comparisons.
- **Memory Complexity:** $O(1)$ additional storage for direction cosines.
- **Strengths:** Significantly faster than the standard Euclidean distance formula, which is critical for interactive rendering of large volume datasets [S1, unofficial].
- **Limitations:** Only applicable to parallel projections. Not valid for perspective projections where rays diverge [S1, unofficial]. Requires precomputation of $d_0$ and direction cosines.
- **When to Use:** Interactive volume rendering using parallel projection and octree-based spatial subdivision.
- **When Not to Use:** Perspective projection, non-octree structures, or when high precision is required for non-axis-aligned cases not covered by the two defined cases.

## Background
This method was introduced in the context of an octree-based multiresolution approach for interactive rendering of very large volume data sets [S1, unofficial]. It addresses the computational bottleneck of calculating ray-box intersection lengths during traversal of the octree structure. The approach leverages the uniformity of parallel rays and the hierarchical nature of the octree to simplify distance calculations [S1, unofficial].

## Sources
[S1] An octree-based multiresolution approach supporting interactive rendering of very large volume data sets, https://escholarship.org/uc/item/44p5z0p4, fetched 2026-10-26 (unofficial)
