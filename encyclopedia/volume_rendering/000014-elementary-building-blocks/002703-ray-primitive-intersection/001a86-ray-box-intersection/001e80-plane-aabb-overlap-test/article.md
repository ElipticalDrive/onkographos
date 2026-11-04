# Plane-AABB Overlap Test
Also called: Plane vs Bounds Intersection, Plane-AABB Intersection Test, Plane-Box Overlap Test, planeBoxOverlapCUDA

## Overview
Determines if an axis-aligned bounding box (AABB) overlaps with a plane defined by a normal vector and a point (or signed distance). It solves the problem of culling voxels or primitives that lie entirely on one side of a cutting plane. It differs from [Ray-box intersection](776-ray-box-intersection.md) by testing a static infinite plane against a static box rather than a parametric ray, and from [AABB Collision Detection](2052-aabb-collision-detection.md) by handling a single infinite half-space rather than two moving boxes.

## How it works
**Inputs:**
- `plane_normal`: A 3D vector $(n_x, n_y, n_z)$ representing the plane's normal.
- `plane_point`: A 3D vector $(p_x, p_y, p_z)$ on the plane, OR `plane_d`: a scalar signed distance from the origin to the plane.
- `aabb_center`: A 3D vector $(c_x, c_y, c_z)$ representing the center of the AABB.
- `aabb_extents`: A 3D vector $(e_x, e_y, e_z)$ representing the half-widths of the AABB along each axis.

**Outputs:**
- Boolean: `true` if the AABB intersects or touches the plane; `false` otherwise.

**Data Structures:**
- `Vector3`: A struct with fields `x`, `y`, `z` of type `float`.
- `Plane`: A struct with fields `normal` (Vector3) and `distance` (float, where $d = -\mathbf{n} \cdot \mathbf{p}$).
- `Bounds`: A struct with fields `center` (Vector3) and `max` (Vector3). Extents are derived as $\mathbf{e} = \mathbf{max} - \mathbf{center}$.

**Formulas:**
1.  **Signed Distance to Center ($s$):** The signed distance from the plane to the AABB center.
    $$s = \mathbf{n} \cdot \mathbf{c} - d$$
    Where $d$ is the plane's distance parameter. If defined by a point $\mathbf{p}$, $d = \mathbf{n} \cdot \mathbf{p}$.
2.  **Projected Radius ($r$):** The maximum extent of the AABB projected onto the plane normal.
    $$r = e_x |n_x| + e_y |n_y| + e_z |n_z|$$
3.  **Overlap Condition:**
    $$|s| \le r$$

**Steps:**
1.  Calculate the signed distance $s$ from the plane to the AABB center using the dot product of the normal and center, minus the plane's distance parameter.
2.  Calculate the projected radius $r$ by summing the products of each AABB extent component and the absolute value of the corresponding normal component.
3.  Return `true` if the absolute value of $s$ is less than or equal to $r$; otherwise, return `false`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plane_aabb_overlap(plane_normal, plane_d, aabb_center, aabb_extents):
    """
    plane_normal: (nx, ny, nz)
    plane_d: float, signed distance from origin to plane
    aabb_center: (cx, cy, cz)
    aabb_extents: (ex, ey, ez)
    """
    # Calculate signed distance from plane to AABB center
    # s = dot(normal, center) - d
    s = plane_normal.x * aabb_center.x + \
        plane_normal.y * aabb_center.y + \
        plane_normal.z * aabb_center.z - plane_d

    # Calculate projected radius of AABB onto plane normal
    # r = ex*|nx| + ey*|ny| + ez*|nz|
    r = (aabb_extents.x * abs(plane_normal.x) +
         aabb_extents.y * abs(plane_normal.y) +
         aabb_extents.z * abs(plane_normal.z))

    # Check if distance is within projected radius
    return abs(s) <= r
```

**Alternative Implementation (Corner Projection):**
Some implementations [S1, S2, unofficial] compute the minimum and maximum projections of the box corners onto the normal explicitly, avoiding absolute values in the radius calculation by selecting corners based on normal signs.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plane_aabb_overlap_corners(plane_normal, plane_d, aabb_center, aabb_extents):
    """
    Alternative method using explicit corner projection.
    Returns true if overlap exists.
    """
    # Determine the corner that minimizes the dot product with the normal
    vmin = [0.0, 0.0, 0.0]
    # Determine the corner that maximizes the dot product with the normal
    vmax = [0.0, 0.0, 0.0]

    for dim in range(3):
        if plane_normal[dim] > 0.0:
            vmin[dim] = aabb_center[dim] - aabb_extents[dim]
            vmax[dim] = aabb_center[dim] + aabb_extents[dim]
        else:
            vmin[dim] = aabb_center[dim] + aabb_extents[dim]
            vmax[dim] = aabb_center[dim] - aabb_extents[dim]

    # Calculate dot products
    dot_vmin = (plane_normal.x * vmin.x +
                plane_normal.y * vmin.y +
                plane_normal.z * vmin.z)
    
    dot_vmax = (plane_normal.x * vmax.x +
                plane_normal.y * vmax.y +
                plane_normal.z * vmax.z)

    # The plane equation is dot(n, x) - d = 0.
    # We check if the range [dot_vmin - d, dot_vmax - d] contains 0.
    # If dot_vmin - d > 0, the entire box is on the positive side (no overlap).
    if dot_vmin - plane_d > 0.0:
        return False
    
    # If dot_vmax - d >= 0, the box intersects or touches the plane.
    if dot_vmax - plane_d >= 0.0:
        return True

    return False
```

**Edge Cases and Pitfalls:**
- **Precision:** When $|s| \approx r$, floating-point inaccuracies may cause false negatives or positives. Using `>=` and `<=` with a small epsilon is recommended for robustness [G].
- **Degenerate Box:** If extents are zero, the test reduces to checking if the center lies on the plane.
- **Normal Normalization:** The normal vector must be normalized for the distance $s$ and radius $r$ calculations to be geometrically correct. If the normal is not normalized, $r$ must be scaled by $1/|\mathbf{n}|$ or $s$ by $|\mathbf{n}|$ [G].

## Subtopics

### [Ray-box intersection](776-ray-box-intersection.md)
The parent method. Use this when you need entry/exit distances for a ray.

### [AABB Culling](1789-aabb-culling.md)
Use this when culling entire boxes against view frustum planes.

### [AABB Collision Detection](2052-aabb-collision-detection.md)
Use this for dynamic collision between two boxes.

### [Ray-Bounding Box Intersection and Sampling Preparation](2364-ray-bounding-box-intersection-and-sampling-preparation.md)
Use this when preparing to sample volume data inside a box.

### [Ray-Cube Intersection](2466-ray-cube-intersection.md)
Use this for specialized cube-shaped primitives.

### [Raycasting Model Matrix & AABB Planes](2607-raycasting-model-matrix-aabb-planes.md)
Use this when transforming rays into local box space.

### [Branchless Ray/AABB Intersection with NaN Handling](3297-branchless-ray-aabb-intersection-with-nan-handling.md)
Use this for GPU shaders requiring branchless execution.

### [2D Axis-Aligned Bounding Box (AABB) Intersection](4027-2d-axis-aligned-bounding-box-aabb-intersection.md)
Use this for 2D projections.

## Operating characteristics
- **Time Complexity:** $O(1)$. Requires a constant number of arithmetic operations (dot products, absolute values, comparisons).
- **Memory Complexity:** $O(1)$. No auxiliary data structures required.
- **Performance:** Extremely fast. Suitable for real-time voxelization and frustum culling. The corner-projection method [S1, S2, unofficial] may be slightly slower due to branching but avoids absolute value instructions on some architectures. The projected-radius method [S4] is branchless and often preferred for GPU implementation.
- **Strengths:** Simple, robust, and efficient. Handles arbitrary plane orientations.
- **Limitations:** Only tests against a single plane. Does not provide intersection points or normals.
- **When to use:** Voxelization of triangles, frustum culling, clipping volumes.
- **When not to use:** When ray-box intersection distances are needed, or when testing against multiple planes (use frustum culling instead).

## Background
The plane-AABB overlap test is a fundamental primitive in computer graphics, particularly in voxelization and collision detection. It is widely used in GPU-based voxelizers [S2, S3, unofficial][S4] to determine which voxels intersect with a triangle's plane. The method is derived from the separating axis theorem (SAT), where the plane normal serves as the separating axis. The projected-radius formulation [S4] is common in Unity-based tools, while the corner-projection method [S1, S2, unofficial] appears in various open-source voxelization libraries.

## Sources
[S1] idealab-isu/GPView, https://github.com/idealab-isu/GPView, fetched 2026-10-26 (unofficial)
[S2] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S3] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
[S4] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
