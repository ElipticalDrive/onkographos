# Azimuthal Hit Detection

Also called: azimuthalHit

## Overview
Azimuthal Hit Detection determines whether a ray intersects an azimuthal section of a spherical voxel grid [S1, unofficial]. An azimuthal section is defined by boundaries in the XZ plane, representing constant azimuthal angles [S1, unofficial]. This method projects the 3D ray and voxel boundaries onto the XZ plane to perform 2D intersection tests, serving as one of the two angular components (along with polar intersection) required to traverse a spherical volume [S1, unofficial]. It differs from Ray-Polar Section Intersection by operating on the XZ plane rather than the XY plane, and from Ray-Ray Intersection by handling curved grid boundaries via planar projection [S1, unofficial].

## How it works
**Inputs:**
- `grid`: The spherical voxel grid object containing bounds, center, and angular resolution [S1, unofficial].
- `ray`: The ray being tested, containing origin, direction, and segment endpoints [S1, unofficial].
- `current_azimuthal_voxel`: The integer index of the current azimuthal voxel being traversed [S1, unofficial].
- `collinear_times`: Output container for collinear intersection times [S1, unofficial].
- `t`: Output parameter for the intersection distance [S1, unofficial].
- `max_t`: Maximum valid intersection distance [S1, unofficial].
- `ray_segment`: Ray segment object with `.P1()` (start point) and `.vector()` (direction vector) methods [S1, unofficial].

**Outputs:**
- Boolean indicating if an azimuthal hit occurred [S1, unofficial].
- Updated `t` and `collinear_times` if a hit is found [S1, unofficial].

**Data Structures:**
- `BoundVec3`: A 3D vector structure where the Y-component is implicitly or explicitly set to 0.0 for azimuthal calculations [S1, unofficial].
- `grid.centerToAzimuthalBound(v)`: Returns a `BoundVec3` representing the boundary vector for voxel `v` [S1, unofficial].
- `grid.pMaxAzimuthal(v)`: Returns a structure with fields `P1` and `P2` representing the azimuthal bounds for voxel `v` [S1, unofficial].

**Steps:**
1. **Define Boundary Points:** Calculate two points, `p_one` and `p_two`, representing the azimuthal boundaries of the current and next voxels in the XZ plane [S1, unofficial].
    - `p_one.x = grid.pMaxAzimuthal(current_azimuthal_voxel).P1`
    - `p_one.y = 0.0`
    - `p_one.z = grid.pMaxAzimuthal(current_azimuthal_voxel).P2`
    - `p_two.x = grid.pMaxAzimuthal(current_azimuthal_voxel + 1).P1`
    - `p_two.y = 0.0`
    - `p_two.z = grid.pMaxAzimuthal(current_azimuthal_voxel + 1).P2`
2. **Define Boundary Vectors:** Obtain pointers to the center-to-bound vectors for the current and next voxels [S1, unofficial].
    - `u_min` points to `grid.centerToAzimuthalBound(current_azimuthal_voxel)`
    - `u_max` points to `grid.centerToAzimuthalBound(current_azimuthal_voxel + 1)`
3. **Calculate Displacement Vectors:** Compute vectors from the ray segment's start point (`ray_segment.P1()`) to the boundary points [S1, unofficial].
    - `w_min = p_one - ray_segment.P1()`
    - `w_max = p_two - ray_segment.P1()`
4. **Compute 2D Cross Products:** Calculate the perpendicular components in the XZ plane using the 2D cross product formula $x_1 z_2 - x_2 z_1$ [S1, unofficial].
    - `perp_uv_min = u_min->x() * ray_segment.vector().z() - u_min->z() * ray_segment.vector().x()`
    - `perp_uv_max = u_max->x() * ray_segment.vector().z() - u_max->z() * ray_segment.vector().x()`
    - `perp_uw_min = u_min->x() * w_min.z() - u_min->z() * w_min.x()`
    - `perp_uw_max = u_max->x() * w_max.z() - u_max->z() * w_max.x()`
    - `perp_vw_min = ray_segment.vector().x() * w_min.z() - ray_segment.vector().z() * w_min.x()`
    - `perp_vw_max = ray_segment.vector().x() * w_max.z() - ray_segment.vector().z() * w_max.x()`
5. **Invoke Generalized Angular Hit Logic:** Pass the computed perpendicular components and ray data to the angular hit handler [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class RaySegment:
    def __init__(self, start, direction):
        self._start = start
        self._dir = direction
    def P1(self):
        return self._start
    def vector(self):
        return self._dir

def azimuthalHit(grid, ray, current_azimuthal_voxel, collinear_times, t, max_t, ray_segment):
    # 1. Define boundary points in XZ plane (y=0)
    p_one = Vec3(grid.pMaxAzimuthal(current_azimuthal_voxel).P1, 0.0, grid.pMaxAzimuthal(current_azimuthal_voxel).P2)
    p_two = Vec3(grid.pMaxAzimuthal(current_azimuthal_voxel + 1).P1, 0.0, grid.pMaxAzimuthal(current_azimuthal_voxel + 1).P2)

    # 2. Get boundary vectors
    u_min = grid.centerToAzimuthalBound(current_azimuthal_voxel)
    u_max = grid.centerToAzimuthalBound(current_azimuthal_voxel + 1)

    # 3. Displacement vectors from ray start
    ray_start = ray_segment.P1()
    ray_vec = ray_segment.vector()
    
    w_min = Vec3(p_one.x - ray_start.x, 0.0, p_one.z - ray_start.z)
    w_max = Vec3(p_two.x - ray_start.x, 0.0, p_two.z - ray_start.z)

    # 4. 2D Cross Products in XZ plane
    # Formula: x1*z2 - x2*z1
    perp_uv_min = u_min.x * ray_vec.z - u_min.z * ray_vec.x
    perp_uv_max = u_max.x * ray_vec.z - u_max.z * ray_vec.x
    
    perp_uw_min = u_min.x * w_min.z - u_min.z * w_min.x
    perp_uw_max = u_max.x * w_max.z - u_max.z * w_max.x
    
    perp_vw_min = ray_vec.x * w_min.z - ray_vec.z * w_min.x
    perp_vw_max = ray_vec.x * w_max.z - ray_vec.z * w_max.x

    # 5. Call generalized angular hit handler
    # Note: The sources do not provide the implementation of angularHit; it handles the actual intersection logic using the computed perpendicular components [S1, unofficial].
    # return angularHit(grid, ray, perp_uv_min, perp_uv_max, perp_uw_min, perp_uw_max, perp_vw_min, perp_vw_max, ray_segment, collinear_times, t, max_t, ray.direction().z(), grid.sphereCenter().z(), grid.pMaxAzimuthal(), current_azimuthal_voxel)
```

**Edge Cases and Pitfalls:**
- **Collinearity:** If the ray is parallel to the azimuthal boundary, the cross products will be zero, and the angular hit handler updates `collinear_times` accordingly [S1, unofficial].
- **Precision:** Floating-point comparisons for equality should use a tolerance epsilon if exact equality is not guaranteed by the grid construction [G].
- **Boundary Conditions:** The method assumes `current_azimuthal_voxel + 1` is a valid index; bounds checking mechanism is not given in the sources.

## Subtopics

### Ray-Polar Section Intersection
Computes intersections in the XY plane for polar angles. Choose when traversing the polar dimension of a spherical grid. [Ray-Polar Section Intersection](3886-ray-polar-section-intersection.md)

### Ray-Ray Intersection
Handles intersections between two rays. Choose when determining ray-ray crossings, not applicable to grid boundary traversal. [Ray-Ray Intersection](7624-ray-ray-intersection.md)

## Operating characteristics
- **Time Complexity**: $O(1)$ per azimuthal step, as it involves a fixed number of arithmetic operations and a single function call [S1, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary space [S1, unofficial].
- **Strengths**: Efficiently reduces 3D spherical intersection to 2D planar geometry and integrates seamlessly with the generalized angular traversal framework [S1, unofficial].
- **Limitations**: Requires accurate pre-computation of azimuthal bounds and performance depends on the underlying angular hit implementation [S1, unofficial].
- **When to use**: During spherical voxel grid traversal when crossing azimuthal boundaries [S1, unofficial].
- **When not to use**: When traversing radial or polar boundaries, or when the grid is not spherical [S1, unofficial].

## Background
The method is part of the Spherical Volume Rendering (SVR) algorithm, implemented in the `svr-algorithm` repository [S1, unofficial]. It follows the mathematical principles of decomposing spherical intersections into planar components for efficient traversal [S1, unofficial].

## Sources
[S1] spherical-volume-rendering/svr-algorithm, https://github.com/spherical-volume-rendering/svr-algorithm, fetched 2026-10-24 (unofficial)
