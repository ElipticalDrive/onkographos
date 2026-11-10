# Ray-Volume Intersection (Face Distance Method)

Also called: Six-face intersection check, Volume::intersect

## Overview
This method determines if a viewing ray intersects the bounding volume of a dataset and calculates the entry point for subsequent volume traversal [S1, unofficial][S2, unofficial]. It serves as the initial geometric culling step in ray casting, ensuring that rays missing the data volume are discarded before expensive sampling occurs [S1, unofficial]. Unlike [Ray-box intersection](776-ray-box-intersection.md), which typically uses slab-based interval intersection for axis-aligned bounding boxes, this method explicitly computes distances to the six faces of the data volume to determine a planar impact coordinate [S1, unofficial]. It is distinct from [Homogeneous Volume Intersection](3260-homogeneous-volume-intersection.md) by not assuming uniform density for analytical free-path calculation, and from [Near/Far Distance Precomputation Pass](3576-near-far-distance-precomputation-pass.md) by using analytical geometry rather than rasterization to find bounds [S1, unofficial][G].

## How it works
**Inputs and Outputs**
*   **Input**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized to unit length) [S1, unofficial]. The volume is defined by its six faces (e.g., a rectangular prism or specific geometry) [S1, unofficial][S2, unofficial].
*   **Output**: Boolean indicating intersection; if intersecting, the entry point (impact point) on the volume surface [S1, unofficial].

**Data Structures**
The volume is represented by its geometric boundaries. For a standard rectangular data volume, this implies six planes defining the faces [S1, unofficial]. The ray is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].

**Steps**
1.  **Check Origin Inside**: Test if the ray origin $\mathbf{o}$ lies within the data volume [S1, unofficial].
    *   If yes, the impact point is set to $\mathbf{o}$ [S1, unofficial].
    *   If no, proceed to step 2.
2.  **Calculate Face Distances**: For each of the six faces of the data volume, calculate the distance from the ray origin to the face [S1, unofficial].
    *   Let $d_i$ be the distance from $\mathbf{o}$ to face $i$.
    *   Let $c_i$ be the corresponding component of the ray direction vector $\mathbf{d}$ associated with that face's normal or axis [S1, unofficial].
    *   Compute the parameter $t_i = d_i / c_i$ [S1, unofficial].
3.  **Compute Planar Impact Coordinates**: Multiply each calculated distance parameter $t_i$ by the corresponding eye positional ordinate (coordinate component) to create a planar impact coordinate [S1, unofficial].
    *   Impact point candidate $\mathbf{p}_i = \mathbf{o} + t_i \mathbf{d}$ [G] (derived from "multiply... by corresponding eye positional ordinate to create a planar impact coordinate" [S1, unofficial]).
4.  **Select Nearest Intersection**:
    *   Filter valid intersections (where $t_i > 0$ and the point lies on the face segment) [G].
    *   Select the smallest valid $t_i$ (or corresponding impact point) as the nearest ray-volume impact point [S1, unofficial].
5.  **Miss Handling**: If no intersection points are found, the ray misses the volume, and processing proceeds to the next pixel [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_volume_intersect_face_distance(ray_origin, ray_direction, volume_faces):
    """
    ray_origin: float3 (x, y, z)
    ray_direction: float3 (normalized)
    volume_faces: list of 6 plane definitions or boundary limits
    Returns: (hit: bool, t_min: float, impact_point: float3)
    """
    t_min = infinity
    hit = False
    
    # Step 1: Check if origin is inside volume
    if is_inside_volume(ray_origin, volume_faces):
        return True, 0.0, ray_origin

    # Step 2 & 3: Calculate distances to each of the six faces
    for face in volume_faces:
        # Calculate distance from ray origin to the face plane
        # For a plane with normal n and point p_on_plane:
        # d = dot(p_on_plane - ray_origin, n)
        # However, source [S1, unofficial] describes dividing distance by ray component.
        # Assuming axis-aligned faces for standard implementation:
        
        # Example for a face perpendicular to X-axis at x = x_limit
        # Distance from origin to plane along X is (x_limit - ray_origin.x)
        # Ray component is ray_direction.x
        # t = (x_limit - ray_origin.x) / ray_direction.x
        
        # Generalizing based on [S1, unofficial]: "distance from each face ... divided by 
        # corresponding component of the ray"
        
        # Calculate t for this face
        t = calculate_face_t(ray_origin, ray_direction, face)
        
        # Step 3: Create planar impact coordinate
        # "Multiply each calculated distance ... by corresponding eye positional ordinate"
        # This implies computing the point on the ray: p = o + t * d
        impact_point = ray_origin + t * ray_direction
        
        # Validate if this intersection is valid (in front of ray, on the face)
        if t > 0 and is_on_face(impact_point, face):
            if t < t_min:
                t_min = t
                hit = True
                best_impact = impact_point

    # Step 5: Miss handling
    if not hit:
        return False, infinity, None
    
    return True, t_min, best_impact

def calculate_face_t(origin, direction, face):
    # Implementation depends on specific face geometry.
    # For axis-aligned box face at limit L along axis A:
    # t = (L - origin[A]) / direction[A]
    # Note: Division by zero if direction[A] is 0 (ray parallel to face)
    # If parallel, no intersection with this specific infinite plane 
    # (or ray lies in it, handled by inside check).
    return (face.limit - origin[face.axis]) / direction[face.axis]

def is_inside_volume(origin, volume_faces):
    # Check if origin is within all 6 boundaries
    for face in volume_faces:
        if not is_within_boundary(origin, face):
            return False
    return True
```

**Parameters**
*   Ray direction normalization: Unit length [S1, unofficial].
*   Volume definition: Six faces [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Ray Parallel to Face**: If the ray direction component corresponding to a face normal is zero, division by zero occurs. Such rays do not intersect that specific infinite plane (unless contained within it, which is handled by the "inside" check) [G].
*   **Miss Handling**: If no valid $t_i$ is found, the ray misses the volume entirely. The algorithm must return a miss status to avoid invalid traversal [S1, unofficial].
*   **Precision**: Floating-point precision issues may cause rays to miss thin volumes or intersect incorrectly near boundaries [G].

## Subtopics

### [Homogeneous Volume Intersection](3260-homogeneous-volume-intersection.md)
Assumes uniform density, allowing direct analytical sampling without majorant-based rejection sampling.

### [Near/Far Distance Precomputation Pass](3576-near-far-distance-precomputation-pass.md)
Uses rasterization to compute entry/exit depths, differing from this analytical face-distance approach[G].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per ray, as it involves a fixed number of face checks (6 for a hexahedron) [S1, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary memory for storing face parameters [G].
*   **Strengths**: Simple, fast culling for rectangular volumes; avoids rasterization overhead [S1, unofficial].
*   **Limitations**: Requires explicit face definitions; less efficient for complex non-convex geometries compared to hierarchical methods [G].
*   **When to Use**: Initial culling for axis-aligned or simple convex volumes in ray casting [S1, unofficial].
*   **When Not to Use**: Complex procedural geometries where [Procedural Ray/Box Intersection](8356-procedural-ray-box-intersection.md) or triangle-based intersection [S2, unofficial] is more appropriate.

## Background
This method originates from early volumetric rendering techniques for displaying 3D aerodynamic flow field data [S1, unofficial]. It was used as a two-step process to determine ray-volume intersection before incremental ray extension through the data [S1, unofficial]. Early versions of VRVE (Volumetric Rendering of Volume Elements) used similar approaches, though they faced aliasing issues with uniform resampling [S1, unofficial]. The method is also referenced in CUDA-based volume rendering implementations [S2, unofficial].

## Sources
[S1] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
[S2] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
