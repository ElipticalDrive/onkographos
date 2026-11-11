# Sparse Brick Set Rendering

Also called: Raytraced SDF rendering with bricks

## Overview
Sparse Brick Set Rendering is a method for rendering Signed Distance Fields (SDFs) using hardware-accelerated raytracing against a Bounding Volume Hierarchy (BVH) of leaf bricks, where each brick contains SDF data and is intersected via software shaders and sphere-tracing [S1, unofficial]. It differs from its parent [ExaBricks Rendering with RTX BVH](2577-exabricks-rendering-with-rtx-bvh.md) by specifically utilizing SDF sphere-tracing within leaf nodes rather than general volume integration, and from [RTX BVH Ray Tracing](2957-rtx-bvh-ray-tracing.md) by relying on software intersection shaders for the SDF geometry rather than hardware-native triangle or primitive intersection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of leaf bricks, each containing a 3D texture of distance values. A BVH constructed from AABBs enclosing these bricks. A camera ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1, unofficial].
*   **Output:** The intersection distance $t$ to the nearest isosurface, the surface normal $\mathbf{n}$, and the object-space position $\mathbf{p}$ [S1, unofficial].

**Data Structures:**
*   **Brick:** A 3D texture of size $8^3 = 512$ samples [S1, unofficial].
    *   **Inner Data:** The inner $6^3$ samples are sphere-traced [S1, unofficial].
    *   **Padding:** A neighbourhood of voxels is duplicated for each brick to avoid the texture sampler crossing brick boundaries during linear filtering [S1, unofficial].
    *   **Note:** 58% of each brick is dedicated to adjacency data [S1, unofficial].
*   **BVH:** A hierarchy of AABBs. Only leaf nodes (representing individual bricks) are stored in the final structure used for traversal; internal nodes are discarded after construction [S1, unofficial].

**Algorithm Steps:**
1.  **BVH Construction:** Construct an AABB for every brick and place them into a BVH for hardware-accelerated raytracing. Only leaf bricks are required; other levels are discarded after construction [S1, unofficial].
2.  **BVH Traversal:** Perform hardware-accelerated ray-BVH traversal. When a ray potentially intersects a leaf brick's AABB, invoke a software intersection shader [S1, unofficial].
3.  **AABB Intersection:** Calculate the entry ($t_{min}$) and exit ($t_{max}$) distances of the ray with respect to the brick's AABB. This defines the ray interval for checking intersection with the isosurface [S1, unofficial].
4.  **Local Coordinate Transformation:** Transform the intersection point into the bounding box's local coordinate system, ranging from $[0, 1]$ [S1, unofficial].
5.  **Sphere-Tracing:** Iteratively find the intersection between the ray and the isosurface using sphere-tracing [S1, unofficial].
6.  **Distance Sampling:** Sample distance values from the volume using trilinear sampling [S1, unofficial].
7.  **Distance Transformation:** Transform sampled distances from their formatted form in the volume texture to distances in the space encoded within the AABB where sphere tracing is performed [S1, unofficial].
8.  **Termination:** Terminate sphere-tracing upon the first intersection with the surface, as all geometry within a brick is opaque [S1, unofficial].
9.  **Object-Space Transformation:** Transform the intersection point back into object-space to calculate the $t$-value of intersection as the distance between the object-space ray origin and the object-space position of intersection [S1, unofficial].
10. **Normal Calculation:** Calculate surface normals using central differencing to find the gradient in the distance field at the point of intersection [S1, unofficial].

**Parameters:**
*   **Brick Size:** $8^3$ voxels. The inner $6^3$ are used for tracing [S1, unofficial].
*   **Sphere-Trace Max Steps:** Not given in the sources.
*   **Epsilon:** Not given in the sources.
*   **Distance Format Transformation:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_brick_set_render(ray_origin, ray_direction, bvh, brick_textures):
    # 1. Hardware BVH traversal to find potential leaf brick intersection
    # Returns brick_id, t_min, t_max of AABB intersection
    brick_id, t_min, t_max = bvh.intersect_aabb(ray_origin, ray_direction)
    if brick_id is None:
        return None

    brick = brick_textures[brick_id]

    # 2. Transform ray to brick local space [0,1]^3
    local_ray_origin = transform_point(ray_origin, M_obj_to_local)
    local_ray_dir = transform_direction(ray_direction, M_obj_to_local)

    # 3. Sphere-tracing in local space
    current_pos = local_ray_origin
    max_steps = 100  # Not given in the sources.
    epsilon = 1e-4   # Not given in the sources.

    for _ in range(max_steps):
        # 4. Sample distance using trilinear filtering
        dist_val = brick.sample_trilinear(current_pos)

        # 5. Transform stored distance value to local space distance
        local_dist = transform_distance_format(dist_val)  # Not given in the sources.

        # 6. Check for intersection
        if local_dist <= epsilon:
            break

        # 7. Advance along the ray
        current_pos = [current_pos[i] + local_ray_dir[i] * local_dist for i in range(3)]

        # Safety check for leaving the brick
        dx = current_pos[0] - local_ray_origin[0]
        dy = current_pos[1] - local_ray_origin[1]
        dz = current_pos[2] - local_ray_origin[2]
        if (dx*dx + dy*dy + dz*dz)**0.5 > t_max:
            return None

    else:
        return None

    # 8. Transform intersection point back to object space
    object_pos = transform_point(current_pos, M_local_to_obj)

    # 9. Calculate t-value
    t = (object_pos[0] - ray_origin[0]) * ray_direction[0] + \
        (object_pos[1] - ray_origin[1]) * ray_direction[1] + \
        (object_pos[2] - ray_origin[2]) * ray_direction[2]

    # 10. Calculate normal using central differencing in object space
    delta = 1.0 / 512.0  # Approximate voxel size in object space
    # Not given in the sources: exact SDF query implementation depends on brick layout.
    dx = (query_sdf(object_pos[0]+delta, object_pos[1], object_pos[2]) - query_sdf(object_pos[0]-delta, object_pos[1], object_pos[2])) / (2 * delta)
    dy = (query_sdf(object_pos[0], object_pos[1]+delta, object_pos[2]) - query_sdf(object_pos[0], object_pos[1]-delta, object_pos[2])) / (2 * delta)
    dz = (query_sdf(object_pos[0], object_pos[1], object_pos[2]+delta) - query_sdf(object_pos[0], object_pos[1], object_pos[2]-delta)) / (2 * delta)
    normal_len = (dx*dx + dy*dy + dz*dz)**0.5
    normal = [dx/normal_len, dy/normal_len, dz/normal_len]

    return t, normal, object_pos

def transform_point(p, M):
    return [p[0]*M[0][0] + p[1]*M[1][0] + p[2]*M[2][0] + M[3][0],
            p[0]*M[0][1] + p[1]*M[1][1] + p[2]*M[2][1] + M[3][1],
            p[0]*M[0][2] + p[1]*M[1][2] + p[2]*M[2][2] + M[3][2]]

def transform_direction(v, M):
    return [v[0]*M[0][0] + v[1]*M[1][0] + v[2]*M[2][0],
            v[0]*M[0][1] + v[1]*M[1][1] + v[2]*M[2][1],
            v[0]*M[0][2] + v[1]*M[1][2] + v[2]*M[2][2]]

def query_sdf(x, y, z): return 0.0  # Not given in the sources.
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Each brick uses $8^3$ samples, with 58% being padding [S1, unofficial].
*   **Performance:** Rendering is generally much faster than construction, and consequently optimization efforts were focused on construction instead [S1, unofficial].
*   **Strengths:** Simple implementation, leverages hardware BVH traversal, avoids boundary issues with texture sampling via padding [S1, unofficial].
*   **Limitations:** High memory overhead due to padding (58%). Sphere-tracing is used for simplicity despite analytical intersections being potentially faster [S1, unofficial]. Central differencing for normals requires 6 samples, whereas the tetrahedron technique requires only 4 [S1, unofficial].

## Background
The method is based on the "sparse brick set" approach described by Evans (2022) [S1, unofficial]. The use of $8^3$ bricks matches the wavefront size of targeted GPU hardware, allowing efficient processing by thread groups of $8^3 = 512$ threads [S1, unofficial]. The rendering technique is similar to hardware-accelerated raytracing methods, but specifically applies to SDFs using software shaders for the intersection logic [S1, unofficial].

## Sources
[S1] Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
