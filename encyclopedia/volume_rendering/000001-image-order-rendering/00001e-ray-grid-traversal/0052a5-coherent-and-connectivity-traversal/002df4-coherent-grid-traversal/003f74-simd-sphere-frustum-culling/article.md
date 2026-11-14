# SIMD Sphere/Frustum Culling

Also called: Sphere/Frustum Culling

## Overview
A culling technique to reject non-overlapping spheres by checking if the sphere center is contained within an enlarged bounding frustum, thereby avoiding unnecessary ray/sphere intersection tests caused by discretization artifacts or spheres smaller than the maximum radius $r_{max}$ [S1]. It sets itself apart from the SIMD shaft culling used in the original Coherent Grid Traversal (CGT) by being appropriate for spheres, which lack the planar edges required for shaft culling [S1].

## How it works
**Inputs:**
*   A packet of rays bounded by an original bounding frustum $\mathcal{F}$.
*   A set of spheres, each defined by a center $\mathbf{c}$ and radius $r$.
*   An enlarged bounding frustum $\mathcal{F}'$, constructed by shifting the planes of $\mathcal{F}$ outward by $r_{max}$ (the maximum possible sphere radius in the dataset) [S1]. This enlarged frustum is typically a by-product of the Sphere-Center Method [S1].

**Outputs:**
*   A subset of spheres that potentially intersect the original frustum $\mathcal{F}$, requiring further ray/sphere intersection testing.

**Data Structures:**
*   The frustum is represented by its six planes. The "shifted bounding planes" from the Sphere-Center method define $\mathcal{F}'$ [S1].

**Steps:**
1.  For each sphere in the current cell or packet:
    a.  Determine if the sphere center $\mathbf{c}$ is contained within the enlarged frustum $\mathcal{F}'$.
    b.  The containment test checks if the distance from $\mathbf{c}$ to any plane of $\mathcal{F}'$ is less than or equal to the sphere's radius $r$ [S1]. Specifically, if the center is not contained by $\mathcal{F}'$, the sphere is culled [S1].
    c.  If the center is contained by $\mathcal{F}'$, the sphere is not culled and an intersection test is required [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_culled(sphere_center, sphere_radius, enlarged_frustum_planes):
    """
    sphere_center: vector (x, y, z)
    sphere_radius: float
    enlarged_frustum_planes: list of 6 plane definitions (normal, distance)
    Returns True if the sphere is culled (no intersection possible).
    """
    for plane in enlarged_frustum_planes:
        # plane.normal is normalized vector (nx, ny, nz)
        # plane.distance is scalar d
        # Distance from point to plane: dot(normal, point) - distance
        dist = dot(plane.normal, sphere_center) - plane.distance
        # If distance > radius, the sphere is outside this plane
        if dist > sphere_radius:
            return True
    return False
```

**Parameters:**
*   $r_{max}$: The maximum radius of any sphere in the dataset, used to compute the enlarged frustum $\mathcal{F}'$ [S1].

**Edge Cases and Pitfalls:**
*   **Small Spheres:** This approach does not cull spheres with radii $r < r_{max}$ that are contained by the enlarged frustum $\mathcal{F}'$ but do not overlap the original frustum $\mathcal{F}$ [S1]. These spheres require an intersection test [S1].
*   **Discretization Artifacts:** Spheres may lie within a cell through which the enlarged frustum passes, but the sphere does not overlap either the original or the enlarged frustum. The culling detects this if the center is outside $\mathcal{F}'$, but if the center is inside $\mathcal{F}'$ due to the expansion, the test proceeds [S1].

## Operating characteristics
*   **Complexity:** The culling check requires one SIMD operation on modern CPUs per sphere, as it involves simple distance-to-plane comparisons [S1].
*   **Performance:** Efficient frustum culling significantly reduces the number of ray/sphere intersection tests. In tests with 16 threads, the number of tests skipped ranged from 38.40% to 81.34% across datasets, improving interactive performance (frame rate) by a factor of 1.17 to 1.68 [S1].
    *   Cylinder: 61.84% skipped, 1.40x speed-up.
    *   JP8: 81.34% skipped, 1.68x speed-up.
    *   Bullet: 67.22% skipped, 1.41x speed-up.
    *   Thunder: 58.34% skipped, 1.40x speed-up.
    *   Foam: 55.61% skipped, 1.47x speed-up.
    *   BulletTorso: 38.40% skipped, 1.17x speed-up [S1].
*   **Strengths:** It cancels the behavior of uniform grids requiring more primitive intersection tests than adaptive structures like kd-trees or BVHs by reducing the number of actual tests performed [S1]. It is simpler than shaft culling and appropriate for spheres [S1].
*   **Limitations:** It cannot cull small spheres ($r < r_{max}$) that are inside the expanded frustum but outside the original frustum [S1].

## Background
The method was introduced in the context of a coherent grid traversal approach for visualizing particle-based simulation data [S1]. It was developed to replace SIMD shaft culling, which relies on planar edges and is thus inappropriate for spheres [S1]. The technique leverages the shifted bounding planes generated by the Sphere-Center Method [S1].

## Sources
[S1] A coherent grid traversal approach to visualizing particle-based simulation data, https://ieeexplore.ieee.org/abstract/document/4293019/, fetched 2026-10-25
