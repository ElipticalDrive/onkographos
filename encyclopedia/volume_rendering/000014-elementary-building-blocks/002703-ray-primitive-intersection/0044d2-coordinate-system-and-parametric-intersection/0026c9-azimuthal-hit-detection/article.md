# Azimuthal Hit Detection

Also called: azimuthalHit

## Overview
Azimuthal Hit Detection determines whether a ray intersects an azimuthal section within a spherical voxel grid. An azimuthal section is defined as a surface segment living in the XZ plane, bounded by radial and polar extents. This method serves as the core primitive intersection test for the azimuthal dimension, complementing radial and polar hit detection. It is distinguished from [Ray-Polar Section Intersection](3886-ray-polar-section-intersection.md) by operating on the XZ plane geometry rather than the XY plane, and from [Ray-Ray Intersection](7624-ray-ray-intersection.md) by specifically handling the curved, sector-based boundaries of spherical coordinates rather than straight line segments.

## How it works
**Inputs:**
- `grid`: The spherical voxel grid object containing bounds, center, and azimuthal section data.
- `ray`: The ray being tested, containing origin, direction, and segment endpoints.
- `current_azimuthal_voxel`: The integer index of the current azimuthal voxel being traversed.
- `ray_segment`: The segment of the ray currently under consideration, with methods `P1()` (start point) and `vector()` (direction vector).
- `collinear_times`: Output container for collinearity parameters.
- `t`: Output intersection distance parameter.
- `max_t`: Maximum allowed intersection distance.

**Data Structures:**
- `BoundVec3`: A 3D vector structure with fields `x`, `y`, `z`.
- `grid.pMaxAzimuthal(v)`: Returns a `BoundVec3` representing the outer boundary of azimuthal voxel `v`.
- `grid.centerToAzimuthalBound(v)`: Returns a pointer to a `BoundVec3` representing the inner boundary relative to the grid center for azimuthal voxel `v`.

**Steps:**
1.  **Calculate Boundary Points:**
    -   Compute `p_one`: A `BoundVec3` where `x = grid.pMaxAzimuthal(current_azimuthal_voxel).P1`, `y = 0.0`, and `z = grid.pMaxAzimuthal(current_azimuthal_voxel).P2`.
    -   Compute `p_two`: A `BoundVec3` where `x = grid.pMaxAzimuthal(current_azimuthal_voxel + 1).P1`, `y = 0.0`, and `z = grid.pMaxAzimuthal(current_azimuthal_voxel + 1).P2`.

2.  **Set Boundary Pointers:**
    -   Set `u_min` pointer to reference `grid.centerToAzimuthalBound(current_azimuthal_voxel)`.
    -   Set `u_max` pointer to reference `grid.centerToAzimuthalBound(current_azimuthal_voxel + 1)`.

3.  **Compute Relative Vectors:**
    -   Calculate `w_min = p_one - ray_segment.P1()`.
    -   Calculate `w_max = p_two - ray_segment.P1()`.

4.  **Compute Perpendicular Components (Cross Products in 2D):**
    The following calculations represent the 2D cross product (determinant) of vectors in the XZ plane, effectively computing the Z-component of the 3D cross product.
    -   `perp_uv_min = u_min->x() * ray_segment.vector().z() - u_min->z() * ray_segment.vector().x()`
    -   `perp_uv_max = u_max->x() * ray_segment.vector().z() - u_max->z() * ray_segment.vector().x()`
    -   `perp_uw_min = u_min->x() * w_min.z() - u_min->z() * w_min.x()`
    -   `perp_uw_max = u_max->x() * w_max.z() - u_max->z() * w_max.x()`
    -   `perp_vw_min = ray_segment.vector().x() * w_min.z() - ray_segment.vector().z() * w_min.x()`
    -   `perp_vw_max = ray_segment.vector().x() * w_max.z() - ray_segment.vector().z() * w_max.x()`

5.  **Invoke Angular Hit Logic:**
    -   Return the result of `angularHit` passing `grid`, `ray`, the six computed perpendicular components (`perp_uv_min`, `perp_uv_max`, `perp_uw_min`, `perp_uw_max`, `perp_vw_min`, `perp_vw_max`), `ray_segment`, `collinear_times`, `t`, `max_t`, `ray.direction().z()`, `grid.sphereCenter().z()`, `grid.pMaxAzimuthal()`, and `current_azimuthal_voxel`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def azimuthalHit(grid, ray, current_azimuthal_voxel, ray_segment, collinear_times, t, max_t):
    # Step 1: Calculate boundary points
    p_one_x = grid.pMaxAzimuthal(current_azimuthal_voxel).P1
    p_one_z = grid.pMaxAzimuthal(current_azimuthal_voxel).P2
    p_one = BoundVec3(x=p_one_x, y=0.0, z=p_one_z)

    p_two_x = grid.pMaxAzimuthal(current_azimuthal_voxel + 1).P1
    p_two_z = grid.pMaxAzimuthal(current_azimuthal_voxel + 1).P2
    p_two = BoundVec3(x=p_two_x, y=0.0, z=p_two_z)

    # Step 2: Set boundary pointers
    u_min = grid.centerToAzimuthalBound(current_azimuthal_voxel)
    u_max = grid.centerToAzimuthalBound(current_azimuthal_voxel + 1)

    # Step 3: Compute relative vectors
    ray_p1 = ray_segment.P1()
    ray_vec = ray_segment.vector()

    w_min_x = p_one.x - ray_p1.x
    w_min_z = p_one.z - ray_p1.z
    w_min = BoundVec3(x=w_min_x, y=0.0, z=w_min_z)

    w_max_x = p_two.x - ray_p1.x
    w_max_z = p_two.z - ray_p1.z
    w_max = BoundVec3(x=w_max_x, y=0.0, z=w_max_z)

    # Step 4: Compute perpendicular components
    perp_uv_min = u_min.x * ray_vec.z - u_min.z * ray_vec.x
    perp_uv_max = u_max.x * ray_vec.z - u_max.z * ray_vec.x

    perp_uw_min = u_min.x * w_min.z - u_min.z * w_min.x
    perp_uw_max = u_max.x * w_max.z - u_max.z * w_max.x

    perp_vw_min = ray_vec.x * w_min.z - ray_vec.z * w_min.x
    perp_vw_max = ray_vec.x * w_max.z - ray_vec.z * w_max.x

    # Step 5: Invoke angular hit logic
    return angularHit(
        grid=grid,
        ray=ray,
        perp_uv_min=perp_uv_min,
        perp_uv_max=perp_uv_max,
        perp_uw_min=perp_uw_min,
        perp_uw_max=perp_uw_max,
        perp_vw_min=perp_vw_min,
        perp_vw_max=perp_vw_max,
        ray_segment=ray_segment,
        collinear_times=collinear_times,
        t=t,
        max_t=max_t,
        ray_dir_z=ray.direction().z(),
        grid_center_z=grid.sphereCenter().z(),
        p_max_azimuthal=grid.pMaxAzimuthal(),
        current_azimuthal_voxel=current_azimuthal_voxel
    )
```

**Edge Cases and Pitfalls:**
-   **Precision:** The perpendicular component calculations involve subtractions of products. In cases where the ray is nearly parallel to the azimuthal boundaries, catastrophic cancellation may occur. [G] Use double precision for all intermediate calculations to maintain accuracy.
-   **Boundaries:** The method assumes `current_azimuthal_voxel + 1` is a valid index. If the ray exits the grid bounds, the caller must ensure this index is within the grid's azimuthal range before calling this function.
-   **Collinearity:** If the ray is collinear with an azimuthal boundary, the perpendicular components will be zero. The `angularHit` function must handle this case to avoid division by zero or incorrect intersection detection.

## Operating characteristics
**Time and Memory Complexity:**
-   Time: $O(1)$ per call, involving a fixed number of arithmetic operations.
-   Memory: $O(1)$ auxiliary space.

**Measured Performance:**
-   Not given in the sources.

**Scaling:**
-   Performance is independent of grid resolution, as it operates on a single voxel's boundary data.

**Strengths and Limitations:**
-   Strengths: Efficient primitive test for azimuthal traversal; minimal computational overhead.
-   Limitations: Relies on the correctness of the underlying `angularHit` function; sensitive to numerical precision in near-collinear cases.

**When to Use:**
-   Use when traversing a spherical voxel grid and needing to determine if a ray crosses an azimuthal boundary.

**When Not to Use:**
-   Do not use for polar or radial boundary checks; use [Ray-Polar Section Intersection](3886-ray-polar-section-intersection.md) or radial hit detection instead.

## Background
Azimuthal Hit Detection is part of the spherical volume rendering algorithm implemented in the `svr-algorithm` repository. It is designed to work within a spherical voxel grid framework where space is partitioned into radial, polar, and azimuthal sections. The method leverages 2D cross-product-like calculations in the XZ plane to determine intersections with azimuthal sectors.

## Sources
[S1] spherical-volume-rendering/svr-algorithm, https://github.com/spherical-volume-rendering/svr-algorithm, fetched 2026-10-24 (unofficial)
