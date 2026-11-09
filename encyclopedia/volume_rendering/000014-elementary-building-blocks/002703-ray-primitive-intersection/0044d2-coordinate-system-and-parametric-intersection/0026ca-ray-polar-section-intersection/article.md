# Ray-Polar Section Intersection

Also called: Polar Hit Detection, polarHit

## Overview
Ray-Polar Section Intersection determines whether a ray intersects a "polar section," which is a volumetric primitive defined within the XY plane of a spherical voxel grid [S1, unofficial]. It serves as the specific implementation for polar coordinates within the broader Ray-Primitive Intersection framework, distinguishing itself from [Azimuthal Hit Detection](3885-azimuthal-hit-detection.md) by operating on the Y-axis plane rather than the Z-axis, and from [Ray-Ray Intersection](7624-ray-ray-intersection.md) by handling planar sector boundaries rather than linear line segments [S1, unofficial].

## How it works
The method computes the intersection of a ray segment with the boundaries of a polar voxel section. It relies on precomputed bounding data for the spherical grid and delegates the core geometric logic to a generalized angular hit detection function.

### Inputs and Outputs
**Inputs:**
*   `grid`: The spherical voxel grid object containing polar boundary data (`pMaxPolar`, `centerToPolarBound`) and geometric constants (`sphereCenter`).
*   `ray`: The ray being tested, providing direction and origin.
*   `ray_segment`: A segment of the ray, providing endpoints `P1()` and `P2()`, and the vector `vector()`.
*   `current_polar_voxel`: The index of the current polar voxel being traversed.
*   `collinear_times`: A container for storing intersection times if the ray is collinear with boundaries.
*   `t`: The current traversal parameter along the ray.
*   `max_t`: The maximum valid traversal parameter.

**Outputs:**
*   Returns an integer indicating the result of the intersection test (typically 0 for no hit, 1 for hit, or specific codes for entry/exit).
*   Updates `t` and `collinear_times` if intersections are found.

### Data Structures
The grid provides specific structures for polar bounds:
*   `grid.pMaxPolar(v)`: Returns a structure with fields `P1` and `P2`. These represent the radial and angular extents of the polar section at voxel index `v`.
*   `grid.centerToPolarBound(v)`: Returns a pointer to a `BoundVec3` structure with fields `x()` and `y()`. These represent the transformed boundary coordinates for the current voxel.

### Steps
1.  **Retrieve Boundary Points:**
    *   Let $p_{one}$ be a 3D vector where $x = \text{grid.pMaxPolar}(\text{current\_polar\_voxel}).P1$, $y = \text{grid.pMaxPolar}(\text{current\_polar\_voxel}).P2$, and $z = 0.0$.
    *   Let $p_{two}$ be a 3D vector where $x = \text{grid.pMaxPolar}(\text{current\_polar\_voxel} + 1).P1$, $y = \text{grid.pMaxPolar}(\text{current\_polar\_voxel} + 1).P2$, and $z = 0.0$.

2.  **Retrieve Bound Vectors:**
    *   Set pointer $u_{min}$ to reference $\text{grid.centerToPolarBound}(\text{current\_polar\_voxel})$.
    *   Set pointer $u_{max}$ to reference $\text{grid.centerToPolarBound}(\text{current\_polar\_voxel} + 1)$.

3.  **Calculate Displacement Vectors:**
    *   Calculate $w_{min} = p_{one} - \text{ray\_segment.P1()}$.
    *   Calculate $w_{max} = p_{two} - \text{ray\_segment.P1()}$.

4.  **Calculate Perpendicular Cross-Products (2D):**
    The algorithm uses a 2D cross-product equivalent for vectors in the XY plane: $\text{perp}(a, b) = a.x \cdot b.y - a.y \cdot b.x$.
    *   Calculate $perp\_uv\_min = u_{min}.x \cdot \text{ray\_segment.vector}.y - u_{min}.y \cdot \text{ray\_segment.vector}.x$.
    *   Calculate $perp\_uv\_max = u_{max}.x \cdot \text{ray\_segment.vector}.y - u_{max}.y \cdot \text{ray\_segment.vector}.x$.
    *   Calculate $perp\_uw\_min = u_{min}.x \cdot w_{min}.y - u_{min}.y \cdot w_{min}.x$.
    *   Calculate $perp\_uw\_max = u_{max}.x \cdot w_{max}.y - u_{max}.y \cdot w_{max}.x$.
    *   Calculate $perp\_vw\_min = \text{ray\_segment.vector}.x \cdot w_{min}.y - \text{ray\_segment.vector}.y \cdot w_{min}.x$.
    *   Calculate $perp\_vw\_max = \text{ray\_segment.vector}.x \cdot w_{max}.y - \text{ray\_segment.vector}.y \cdot w_{max}.x$.

5.  **Delegate to Angular Hit Function:**
    *   Return the result of `angularHit` passing the grid, ray, all calculated perpendicular values, the ray segment, collinear times, $t$, $max\_t$, the ray's Y-direction component, the grid's sphere center Y-coordinate, the grid's polar max data, and the current polar voxel index.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_polar_section_intersection(grid, ray, ray_segment, current_polar_voxel, 
                                   collinear_times, t, max_t):
    # Step 1: Retrieve boundary points from grid
    p_max_curr = grid.pMaxPolar(current_polar_voxel)
    p_max_next = grid.pMaxPolar(current_polar_voxel + 1)
    
    p_one = Vec3(x=p_max_curr.P1, y=p_max_curr.P2, z=0.0)
    p_two = Vec3(x=p_max_next.P1, y=p_max_next.P2, z=0.0)
    
    # Step 2: Retrieve bound vectors
    u_min_ptr = grid.centerToPolarBound(current_polar_voxel)
    u_max_ptr = grid.centerToPolarBound(current_polar_voxel + 1)
    
    # Step 3: Calculate displacement vectors
    ray_p1 = ray_segment.P1()
    ray_vec = ray_segment.vector()
    
    w_min = p_one - ray_p1
    w_max = p_two - ray_p1
    
    # Step 4: Calculate perpendicular cross-products (2D)
    # perp(a, b) = a.x * b.y - a.y * b.x
    
    perp_uv_min = u_min_ptr.x * ray_vec.y - u_min_ptr.y * ray_vec.x
    perp_uv_max = u_max_ptr.x * ray_vec.y - u_max_ptr.y * ray_vec.x
    
    perp_uw_min = u_min_ptr.x * w_min.y - u_min_ptr.y * w_min.x
    perp_uw_max = u_max_ptr.x * w_max.y - u_max_ptr.y * w_max.x
    
    perp_vw_min = ray_vec.x * w_min.y - ray_vec.y * w_min.x
    perp_vw_max = ray_vec.x * w_max.y - ray_vec.y * w_max.x
    
    # Step 5: Delegate to angular hit function
    # Note: The specific implementation of angularHit is not detailed in the 
    # provided source notes, but it handles the core intersection logic.
    result = angularHit(
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
        ray_dir_y=ray.direction().y(),
        sphere_center_y=grid.sphereCenter().y(),
        p_max_polar=grid.pMaxPolar(),
        current_polar_voxel=current_polar_voxel
    )
    
    return result
```

### Parameters and Edge Cases
*   **Precision:** The calculation involves differences of coordinates and cross-products. Floating-point precision errors may occur if the ray is nearly parallel to the polar section boundaries. The source notes mention a separate handling for collinear cases via `collinear_times` [S1, unofficial].
*   **Boundaries:** The method accesses `current_polar_voxel + 1`. Care must be taken to ensure `current_polar_voxel + 1` is within the valid range of the grid's polar sections to avoid out-of-bounds access.
*   **Degenerate Input:** If the ray segment has zero length, the vector calculation will be zero, potentially leading to division by zero or incorrect intersection results in the delegated `angularHit` function.

## Operating characteristics
*   **Time Complexity:** The intersection calculation itself is $O(1)$, involving a fixed number of arithmetic operations. The complexity depends on the delegated `angularHit` function, which likely involves logarithmic or constant time lookups depending on the grid structure.
*   **Memory Complexity:** $O(1)$ auxiliary space, as it only stores a few temporary vectors and scalars.
*   **Strengths:** Efficiently handles the specific geometry of polar sections in spherical grids. Generalizes the logic for both polar and azimuthal cases.
*   **Limitations:** Relies on the correctness and performance of the `angularHit` function. Requires precomputed grid boundary data.

## Background
The Ray-Polar Section Intersection method is part of the Spherical Volume Rendering (SVR) algorithm, specifically designed for rendering data on spherical voxel grids [S1, unofficial]. It was developed as part of the open-source SVR project to provide efficient ray-primitive intersection capabilities for spherical coordinate systems. The method generalizes the intersection logic used for azimuthal sections by adapting the plane of operation to the XY plane for polar sections.

## Sources
[S1] spherical-volume-rendering/svr-algorithm, https://github.com/spherical-volume-rendering/svr-algorithm, fetched 2026-10-24 (unofficial)
