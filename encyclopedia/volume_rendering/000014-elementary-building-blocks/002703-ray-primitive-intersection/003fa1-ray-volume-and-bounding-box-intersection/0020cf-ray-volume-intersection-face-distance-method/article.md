# Ray-Volume Intersection (Face Distance Method)

Also called: Six-face intersection check, Volume::intersect

## Overview
The Ray-Volume Intersection (Face Distance Method) is a two-step procedure to determine if a viewing ray intersects the data volume and to find the entry point for volume traversal [S1, unofficial]. It calculates distances from the ray origin to each of the six faces of the volume geometry, divides by the corresponding ray direction component, and selects the smallest valid distance as the nearest impact point [S1, unofficial].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o} = (o_x, o_y, o_z)$ [S1, unofficial].
*   Ray direction $\mathbf{d} = (d_x, d_y, d_z)$, normalized to unit length [S1, unofficial].
*   Volume geometry definition (typically a rectangular prism defined by six faces, or specific primitives like cylinders/spheres as handled by helper functions [S2, unofficial]).

**Outputs:**
*   Entry distance $t_{in}$ (or impact point) [S1, unofficial].
*   Exit distance $t_{out}$ (or impact point) [S2, unofficial].
*   Boolean flag indicating intersection success [G].

**Data Structures:**
*   The volume is treated as a closed boundary defined by six planes [S1, unofficial].
*   Variables `inAbs` and `outAbs` store the absolute distances along the ray vector [S2, unofficial].

**Steps:**
1.  **Origin Check:** Test if the ray origin $\mathbf{o}$ lies inside the volume boundary [S1, unofficial]. If inside, set the impact point to the ray's origin ($t_{in} = 0$) [S1, unofficial].
2.  **Face Distance Calculation:** If outside, calculate the distance $d_i$ from $\mathbf{o}$ to each of the six faces [S1, unofficial]. Divide $d_i$ by the corresponding component of the ray direction vector $\mathbf{d}$ to obtain a parameter $t_i = d_i / d_{\text{axis}}$ [S1, unofficial].
3.  **Planar Coordinate Mapping:** Multiply each calculated distance parameter $t_i$ by the corresponding eye positional ordinate $e_i$ to create a planar impact coordinate $p_i = t_i \cdot e_i$ [S1, unofficial].
4.  **Selection:** Select the smallest valid distance/coordinate as the nearest ray-volume impact point [S1, unofficial]. If no intersection points are found, the ray misses the volume [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_volume_intersect_face_distance(ray_origin, ray_direction, volume_bounds, eye_ordinates):
    """
    ray_origin: tuple (ox, oy, oz)
    ray_direction: tuple (dx, dy, dz), assumed normalized
    volume_bounds: object with attributes xmin, xmax, ymin, ymax, zmin, zmax
    eye_ordinates: tuple (ex, ey, ez) corresponding to spatial axes
    Returns: (hit, t_entry, t_exit)
    """
    ox, oy, oz = ray_origin
    dx, dy, dz = ray_direction
    ex, ey, ez = eye_ordinates

    # Step 1: Check if origin is inside
    if (volume_bounds.xmin <= ox <= volume_bounds.xmax) and \
       (volume_bounds.ymin <= oy <= volume_bounds.ymax) and \
       (volume_bounds.zmin <= oz <= volume_bounds.zmax):
        return (True, 0.0, 0.0)

    candidates = []
    # Step 2 & 3: Face distance calculation and planar coordinate mapping
    # For each of the six faces, compute t = distance / direction_component
    # Then compute planar coordinate = t * eye_ordinate_component
    faces = [
        (volume_bounds.xmin, dx, ex), (volume_bounds.xmax, dx, ex),
        (volume_bounds.ymin, dy, ey), (volume_bounds.ymax, dy, ey),
        (volume_bounds.zmin, dz, ez), (volume_bounds.zmax, dz, ez)
    ]
    origins = [ox, ox, oy, oy, oz, oz]

    for i, (face_val, dir_comp, eye_comp) in enumerate(faces):
        if dir_comp == 0.0:
            continue # [G] Skip division by zero for rays parallel to face normals
        dist = face_val - origins[i]
        t = dist / dir_comp
        planar_coord = t * eye_comp
        candidates.append((planar_coord, t))

    # Step 4: Select smallest valid distance/coordinate
    # [G] Valid implies t > 0 and intersection lies within face bounds
    valid = [(p, t) for p, t in candidates if t > 0.0]
    if not valid:
        return (False, 0.0, 0.0)

    t_entry = min(t for p, t in valid)
    # [G] Exit distance typically computed via a second pass or traversal bounds
    return (True, t_entry, 0.0)
```

**Edge Cases and Pitfalls:**
*   **Ray Miss:** If no intersection points are found, the ray misses the volume entirely and processing must proceed to the next pixel without traversal [S1, unofficial].
*   **Zero Direction Component:** Division by zero occurs if $d_x, d_y,$ or $d_z$ is zero [G]. The implementation must skip the corresponding face checks or treat them as infinite bounds [G].
*   **Precision:** Floating-point precision errors may cause rays to slightly miss faces or enter/exit incorrectly [G]. Epsilon adjustments may be needed [G].

## Subtopics

### [Homogeneous Volume Intersection](3260-homogeneous-volume-intersection.md)
Homogeneous Volume Intersection computes the first interaction point of a ray within a constant-density volume, or determines if the ray exits the medium without interacting. It differs from [Heterogeneous Volume Intersection](3261-heterogeneous-volume-intersection.md) by assuming a uniform density field, allowing for direct analytical sampling of the free path length without the overhead of majorant-based rejection sampling. It serves as the core traversal logic for participating media where density does not vary spatially.

### [Near/Far Distance Precomputation Pass](3576-near-far-distance-precomputation-pass.md)
The Near/Far Distance Precomputation Pass renders voxel bricks to a render target to compute the entry ($t_{Near}$) and exit ($t_{Far}$) distances of viewing rays intersecting the volume for each pixel. This prepares the bounds for the subsequent raymarching pass, allowing the main rendering pass to skip empty space and focus computation only on occupied regions. It differs from [Homogeneous Volume Intersection](3260-homogeneous-volume-intersection.md) by explicitly rasterizing the volume geometry to derive depth bounds rather than analytically solving for intersections with a homogeneous bounding box.

## Operating characteristics
Not given in the sources.

## Background
The method is described in a DTIC report (ADA202661) detailing volumetric rendering techniques for displaying three-dimensional aerodynamic flow field data [S1, unofficial]. The approach involves a two-step process: first checking if the ray origin is inside the data volume, and if not, calculating distances to the six faces to find the nearest impact point [S1, unofficial]. This method was used to initialize the ray traversal for volume rendering, preceding the actual resampling or marching steps [S1, unofficial]. Later implementations, such as those in CUDA-based volume rendering, adapted this logic, sometimes extending it to handle cylindrical or spherical geometries via specific helper functions (`rayIntersectCyl`, `rayIntersectSpher`) [S2, unofficial].

## Sources
[S1] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
[S2] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
