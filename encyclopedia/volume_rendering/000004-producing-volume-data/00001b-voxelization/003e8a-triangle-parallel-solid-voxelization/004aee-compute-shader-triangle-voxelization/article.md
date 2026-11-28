# Compute Shader Triangle Voxelization

Also called: Large Triangle DDA Voxelization

## Overview
A compute shader technique for voxelizing large triangles by iterating over a 2D bounding box in the plane of the triangle and testing for collisions, designed to handle triangles that span many voxels more efficiently than per-voxel rasterization [S1, unofficial]. It differs from standard rasterization-based voxelization by explicitly solving the plane equation for every candidate voxel within the triangle's 2D footprint, allowing for precise solid voxelization and texture coordinate interpolation without relying on hardware rasterization pipelines [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A buffer of triangle indices (`large_triangles`), global vertex arrays (`vertices`), index arrays (`indices`), a texture map (`triangle_map`), a Uniform Buffer Object (`ubo`) containing AABB bounds (`aabb_min`, `aabb_max`) and image size, and a texture atlas for diffuse colors [S1, unofficial].
*   **Output:** A 3D `voxelTexture` (storage image) where voxel colors are written [S1, unofficial].

**Data Structures:**
*   `large_triangles`: A buffer or array where `large_triangles[gl_WorkGroupID.x].triangle_index` stores the index of the triangle assigned to the workgroup [S1, unofficial].
*   `vertices`: Global array of vertex positions [S1, unofficial].
*   `indices`: Global array of triangle indices [S1, unofficial].
*   `triangle_map`: Array mapping triangle index to texture atlas index [S1, unofficial].
*   `ubo`: Uniform buffer containing `aabb_min` (vec4), `aabb_max` (vec4), and implicitly the voxel grid resolution via `imageSize(voxelTexture).x` [S1, unofficial].
*   `shared uint triangle_index`: Shared memory variable used to broadcast the triangle index from thread 0 to all threads in the workgroup [S1, unofficial].

**Parameters:**
*   `NUM_THREADS`: The number of threads per workgroup is set to 64 for the base method, while the Correct Texture Coordinates variant uses 32 threads per workgroup [S1, unofficial].

**Steps:**
1.  **Triangle Index Broadcast:** Thread 0 loads the triangle index from `large_triangles[gl_WorkGroupID.x].triangle_index` into the shared memory variable `triangle_index`, then a barrier synchronizes all threads [S1, unofficial].
2.  **Vertex Retrieval:** All threads retrieve the three vertex positions from `vertices` using the shared `triangle_index` and the `indices` array, and get the texture index from `triangle_map[triangle_index]` [S1, unofficial].
3.  **Coordinate Transformation:** Transform vertices to world space using `pc.model`, then convert to voxel space using `_min` and `voxel_width` derived from `ubo.aabb_min`, `ubo.aabb_max`, and `imageSize(voxelTexture).x` [S1, unofficial].
4.  **Bounding Box Calculation:** Calculate the bounding box of the triangle in world space (min_x, max_x, etc.) and determine dimensions x_dim, y_dim, z_dim [S1, unofficial].
5.  **Plane Equation:** Compute the plane equation in voxel space: normal via cross product of edge vectors, then coefficients `plane_x_coefficient`, `plane_y_coefficient`, and `plane_constant` derived from the normal and distance D [S1, unofficial].
6.  **2D Bounding Box:** Determine the 2D bounding box in voxel coordinates for the `x` and `y` axes (`min_x_voxel`, `max_x_voxel`, etc.) and calculate total voxel count [S1, unofficial].
7.  **Voxel Iteration and Z-Computation:** Calculate the `z` coordinate on the triangle plane for the current `(i, j)` position using the plane equation coefficients [S1, unofficial].
8.  **Collision Test:** Construct the voxel coordinate `voxel_coord` using `i`, `j`, and the calculated `z_value`, then perform a collision test `voxel_triangle_collision_test` [S1, unofficial].
9.  **Texture Interpolation:** Compute barycentric coordinates in voxel space and interpolate texture coordinates to sample the diffuse color from the texture atlas [S1, unofficial].
10. **Output:** Store the resulting voxel color (diffuse + alpha 1.0) into the `voxelTexture` at `voxel_coord` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_triangle(triangle_index, large_triangles, vertices, indices, triangle_map, ubo, voxel_texture):
    # Thread 0 loads triangle index into shared memory (simulated)
    shared_triangle_index = large_triangles[triangle_index].triangle_index
    # Barrier implied for shared memory sync

    # Retrieve vertices and texture index
    v0 = vertices[indices[shared_triangle_index * 3]]
    v1 = vertices[indices[shared_triangle_index * 3 + 1]]
    v2 = vertices[indices[shared_triangle_index * 3 + 2]]
    tex_idx = triangle_map[shared_triangle_index]

    # Transform to world space
    v0_w = ubo.model_matrix * v0
    v1_w = ubo.model_matrix * v1
    v2_w = ubo.model_matrix * v2

    # Convert to voxel space
    voxel_width = (ubo.aabb_max.x - ubo.aabb_min.x) / ubo.image_size_x
    v0_v = (v0_w - ubo.aabb_min) / voxel_width
    v1_v = (v1_w - ubo.aabb_min) / voxel_width
    v2_v = (v2_w - ubo.aabb_min) / voxel_width

    # Bounding box in voxel space
    min_x = min(v0_v.x, v1_v.x, v2_v.x)
    max_x = max(v0_v.x, v1_v.x, v2_v.x)
    min_y = min(v0_v.y, v1_v.y, v2_v.y)
    max_y = max(v0_v.y, v1_v.y, v2_v.y)
    min_z = min(v0_v.z, v1_v.z, v2_v.z)
    max_z = max(v0_v.z, v1_v.z, v2_v.z)

    # Plane equation in voxel space
    edge1 = v1_v - v0_v
    edge2 = v2_v - v0_v
    normal = cross(edge1, edge2)
    D = dot(normal, v0_v)
    plane_x = normal.x
    plane_y = normal.y
    plane_z = normal.z
    plane_constant = D

    # 2D bounding box for iteration
    min_x_v = int(floor(min_x))
    max_x_v = int(ceil(max_x))
    min_y_v = int(floor(min_y))
    max_y_v = int(ceil(max_y))

    # Thread mapping to 2D grid
    local_thread_id = gl_LocalInvocationID.x  # Standard GPU invocation ID [G]
    width = max_x_v - min_x_v + 1
    height = max_y_v - min_y_v + 1
    if local_thread_id < width * height:
        i = local_thread_id % width + min_x_v
        j = local_thread_id // width + min_y_v

        # Z coordinate on plane
        if plane_z != 0:
            z_val = (plane_constant - plane_x * i - plane_y * j) / plane_z
        else:
            z_val = 0.0  # Degenerate plane parallel to Z-axis [G]

        voxel_coord = vec3(i, j, z_val)

        # Collision test
        if point_in_triangle(voxel_coord, v0_v, v1_v, v2_v):
            # Barycentric coordinates [G]
            bary = compute_barycentric(voxel_coord, v0_v, v1_v, v2_v)
            # Interpolate UVs (assuming u0, u1, u2 are available) [G]
            uv = bary.x * u0 + bary.y * u1 + bary.z * u2
            # Sample color [G]
            color = sample_texture_atlas(tex_idx, uv)
            # Store output
            voxel_texture[voxel_coord] = vec4(color, 1.0)
```

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** If the triangle is degenerate (area zero), the normal will be zero, leading to division by zero in the plane equation [G].
*   **Plane Parallel to Z-axis:** If `plane_z_coefficient` is zero, the plane is vertical relative to the voxel grid's Z-axis, requiring special handling [G].
*   **Precision:** Floating-point precision issues may arise when calculating the plane equation and barycentric coordinates for large triangles or small voxel sizes [G].
*   **Boundary Conditions:** The bounding box calculation must correctly handle voxels that are partially covered by the triangle, and the collision test must be robust to handle edge cases where the voxel center is exactly on the triangle edge [G].

## Subtopics

### Correct Texture Coordinates Voxelization
A compute shader variant for voxelizing triangles with correct texture coordinate interpolation, processing one triangle per thread group (or per invocation depending on launch configuration), focusing on accurate UV mapping [S1, unofficial]. It differs from the Large Triangle DDA method by explicitly calculating and interpolating texture coordinates for each voxel, ensuring that the resulting voxel colors accurately reflect the original triangle's texture [S1, unofficial]. It uses a local size of 32 threads per workgroup [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot A)$, where $N$ is the number of triangles and $A$ is the average area of the triangle in voxels, as each triangle is processed by a workgroup and each thread processes a voxel within the triangle's 2D bounding box [S1, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the number of voxels in the output volume, with minimal shared memory usage limited to storing the triangle index [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with the number of triangles due to independent processing, though performance may be limited by memory bandwidth when writing to the `voxelTexture` [S1, unofficial].
*   **Strengths:** Handles large triangles efficiently by avoiding per-voxel rasterization overhead and provides precise control over voxelization and texture interpolation [S1, unofficial].
*   **Limitations:** Requires careful handling of degenerate cases, precision issues, and vertical planes [G].
*   **When to Use:** Suitable for applications requiring high-quality solid voxelization with accurate texture mapping, such as voxel cone tracing or global illumination [S1, unofficial].
*   **When Not to Use:** Not suitable for real-time applications with extremely large scenes where performance is critical and approximate voxelization is acceptable [G].

## Background
Not given in the sources.

## Sources
[S1] Oshanath/Voxel-Cone-Tracing, https://github.com/Oshanath/Voxel-Cone-Tracing, fetched 2026-10-26 (unofficial)
