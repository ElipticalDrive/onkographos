# Axis-Aligned Projection via Face Normal

Also called: Face Normal Axis Selection, Optimal Axis Projection

## Overview
This method is a geometry shader technique used in rasterization-based voxelization to project 3D mesh triangles onto a 2D plane aligned with the axis most perpendicular to the triangle's face normal [S1, unofficial]. By selecting the optimal viewing axis (X, Y, or Z) for each triangle, it minimizes projection overlap and distortion, ensuring that the triangle's area is preserved as much as possible in the chosen projection plane [S1, unofficial]. This approach is distinct from fixed-axis projection methods by adapting the projection direction per primitive, which is particularly useful for reducing artifacts in voxel grids where triangles might otherwise be foreshortened or occluded incorrectly [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle strip consisting of vertices, each with a world-space position, a normal vector, and texture UV coordinates.
*   **Output:** A geometry stream containing transformed clip-space positions, world-space positions, and texture UVs, ready for rasterization into a voxel texture or buffer.

**Data Structures:**
*   **Vertex Input:** Contains `vertex.xyz` (world position), `normal` (world normal), and `uv` (texture coordinates).
*   **Geometry Output:** Contains `positionCS` (clip space position), `positionWS` (world space position), and `uv` (texture coordinates).

**Steps:**
1.  **Calculate Face Normal:** Compute the average of the three input vertex normals and normalize the result to obtain the triangle's face normal.
    $$ \mathbf{n}_{face} = \text{normalize}(\mathbf{n}_0 + \mathbf{n}_1 + \mathbf{n}_2) $$
2.  **Determine Optimal Axis:** Calculate the absolute dot product of the face normal with the three world-space axes ($\mathbf{x}=(1,0,0)$, $\mathbf{y}=(0,1,0)$, $\mathbf{z}=(0,0,1)$). Select the axis with the maximum dot product.
    $$ d_x = |\mathbf{n}_{face} \cdot \mathbf{x}|, \quad d_y = |\mathbf{n}_{face} \cdot \mathbf{y}|, \quad d_z = |\mathbf{n}_{face} \cdot \mathbf{z}| $$
    The axis index is determined as follows:
    *   If $d_x$ is maximum, select the X-axis (projection onto the ZY plane).
    *   If $d_y$ is maximum, select the Y-axis (projection onto the XZ plane).
    *   If $d_z$ is maximum, select the Z-axis (projection onto the XY plane).
3.  **Transform to Clip Space:** For each vertex, transform its world-space position to clip space.
    $$ \mathbf{p}_{clip} = \text{TransformWorldToHClip}(\mathbf{p}_{world}) $$
4.  **Normalize to NDC:** Divide the clip space position by its w-component to normalize to Normalized Device Coordinates (NDC).
    $$ \mathbf{p}_{ndc} = \frac{\mathbf{p}_{clip}}{w} $$
5.  **Adjust Z Range (Platform Specific):** If the platform uses a 0-1 Z range for clip space (e.g., DirectX), convert the Z coordinate from 0-1 to -1-1 range. This is typically checked using a platform-specific constant like `UNITY_NEAR_CLIP_VALUE`.
    $$ \text{if } \text{UNITY\_NEAR\_CLIP\_VALUE} \ge 0: \quad z_{ndc} = z_{ndc} \times 2.0 - 1.0 $$
6.  **Permute Coordinates:** Swap the XYZ coordinates of the NDC position to align the selected projection axis with the depth (Z) axis for rasterization.
    *   If X-axis is dominant (Index 0): Swap to ZY. $\mathbf{p}_{permuted} = (z, y, x)$? No, the source says `positionCS.xyz = positionCS.zyx`. This maps original X to Z, Y to Y, Z to X? Let's re-read carefully.
        *   Source: `if (axisIdx == 0) ... positionCS.xyz = positionCS.zyx;`
        *   Axis 0 is X-axis dominant. The projection plane is ZY. The depth axis for rasterization should be the one we are projecting *onto*? No, we are projecting *along* the X axis. So the depth buffer should represent X. But the source permutes to `zyx`.
        *   Let's trace: Original $(x, y, z)$. Permute to $(z, y, x)$. The new Z is the old X. This makes the old X the depth axis. This aligns with projecting along the X axis (where X is the depth).
        *   If Y-axis is dominant (Index 1): Swap to XZ. `positionCS.xyz = positionCS.xzy`. Original $(x, y, z) \to (x, z, y)$. New Z is old Y. Depth is Y.
        *   If Z-axis is dominant (Index 2): No swap. Original $(x, y, z) \to (x, y, z)$. New Z is old Z. Depth is Z.
    *   This permutation ensures that the axis of projection (the one with the highest normal alignment) becomes the depth axis (Z) in the clip space, allowing standard rasterization to accumulate coverage along that axis.
7.  **Restore Z Range (Platform Specific):** If the platform uses a 0-1 Z range, convert the Z coordinate back from -1-1 to 0-1.
    $$ \text{if } \text{UNITY\_NEAR\_CLIP\_VALUE} \ge 0: \quad z_{permuted} = z_{permuted} \times 0.5 + 0.5 $$
8.  **De-normalize:** Multiply the permuted position by w to restore the clip space scale.
    $$ \mathbf{p}_{final} = \mathbf{p}_{permuted} \times w $$
9.  **Output:** Output the final clip space position, the original world space position, and the texture UV to the geometry stream. Restart the triangle strip.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_triangle(input_vertices):
    # Step 1: Calculate Face Normal
    n0 = input_vertices[0].normal
    n1 = input_vertices[1].normal
    n2 = input_vertices[2].normal
    face_normal = normalize(n0 + n1 + n2)
    
    # Step 2: Determine Optimal Axis
    axis_x = float3(1.0, 0.0, 0.0)
    axis_y = float3(0.0, 1.0, 0.0)
    axis_z = float3(0.0, 0.0, 1.0)
    
    dot_x = abs(dot(face_normal, axis_x))
    dot_y = abs(dot(face_normal, axis_y))
    dot_z = abs(dot(face_normal, axis_z))
    
    # Select axis index: 0 for X, 1 for Y, 2 for Z
    if dot_x >= dot_y and dot_x >= dot_z:
        axis_idx = 0
    elif dot_y >= dot_x and dot_y >= dot_z:
        axis_idx = 1
    else:
        axis_idx = 2
        
    output_stream = []
    
    for i in range(3):
        vertex = input_vertices[i]
        
        # Step 3: Transform to Clip Space
        position_cs = transform_world_to_hclip(vertex.vertex.xyz)
        
        # Step 4: Normalize to NDC
        w = position_cs.w
        position_ndc = position_cs.xyz / w
        
        # Step 5: Adjust Z Range (Platform Specific)
        # UNITY_NEAR_CLIP_VALUE >= 0 implies 0-1 Z range (e.g., DirectX)
        if UNITY_NEAR_CLIP_VALUE >= 0:
            position_ndc.z = position_ndc.z * 2.0 - 1.0
            
        # Step 6: Permute Coordinates
        if axis_idx == 0: # X-axis dominant -> Project onto ZY, Depth is X
            # Swap to ZY: x->z, y->y, z->x? No, source says zyx
            # Original: (x, y, z) -> New: (z, y, x)
            # New Z is old X.
            position_permuted = float3(position_ndc.z, position_ndc.y, position_ndc.x)
        elif axis_idx == 1: # Y-axis dominant -> Project onto XZ, Depth is Y
            # Swap to XZ: x->x, y->z, z->y? Source says xzy
            # Original: (x, y, z) -> New: (x, z, y)
            # New Z is old Y.
            position_permuted = float3(position_ndc.x, position_ndc.z, position_ndc.y)
        else: # Z-axis dominant -> Project onto XY, Depth is Z
            # No swap
            position_permuted = position_ndc
            
        # Step 7: Restore Z Range (Platform Specific)
        if UNITY_NEAR_CLIP_VALUE >= 0:
            position_permuted.z = position_permuted.z * 0.5 + 0.5
            
        # Step 8: De-normalize
        position_final = position_permuted * w
        
        # Step 9: Output
        output_vertex = {
            'positionCS': position_final,
            'positionWS': vertex.vertex.xyz,
            'uv': transform_tex(vertex.uv, _BaseMap)
        }
        output_stream.append(output_vertex)
        
    output_stream.restart_strip()
    return output_stream
```

**Parameters:**
*   `UNITY_NEAR_CLIP_VALUE`: A platform-specific constant. If $\ge 0$, the platform uses a 0-1 Z range for clip space (e.g., DirectX), requiring conversion to/from -1-1. If $< 0$, it uses a -1-1 range (e.g., OpenGL/Vulkan) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Visual Incorrectness:** The geometry shader output is visually incorrect if rendered directly to a standard screen because the coordinate permutation is intended for rasterization into a specific texture layout (voxel grid), not for standard screen projection [S1, unofficial]. The technique relies on the pixel shader running on the permuted data, not on the visual correctness of the geometry shader output [S1, unofficial].
*   **Degenerate Triangles:** Triangles with zero area or collinear vertices may produce undefined normals. The source does not specify handling for this, but standard geometry shader practices should include a check for degenerate triangles to avoid artifacts [G].
*   **Precision:** The normalization and dot product calculations are sensitive to floating-point precision. Ensure that the normal calculation is robust for nearly coplanar or degenerate triangles [G].

## Operating characteristics
*   **Time Complexity:** The method adds a constant amount of computation per triangle in the geometry shader. The complexity is $O(N)$ where $N$ is the number of triangles, as each triangle is processed independently to determine its projection axis.
*   **Memory Complexity:** The method requires storing the transformed positions and attributes for the output stream. The memory usage is proportional to the number of vertices in the output stream, which is $3 \times N$ for $N$ triangles.
*   **Measured Performance:** Performance depends on the efficiency of the geometry shader and the rasterization stage. The source does not provide specific performance metrics, but the method is designed to be efficient by avoiding complex ray-casting or marching cubes algorithms [S1, unofficial].
*   **Strengths:** Reduces projection distortion and overlap by adapting to the triangle's orientation. Simple to implement within a geometry shader pipeline.
*   **Limitations:** Relies on rasterization, which may have limitations in handling complex overlaps or transparency. The visual output is not directly interpretable as a standard 3D view [S1, unofficial].
*   **When to Use:** Suitable for real-time voxelization of meshes where projection quality is important and the target is a voxel grid rather than a standard screen image.
*   **When Not to Use:** Not suitable for offline rendering or when a standard 3D view is required directly from the geometry shader output.

## Background
This technique is described in a Unity-focused blog post by "seyakara" [S1, unofficial]. It is part of a broader effort to perform real-time GPU-based voxelization of 3D models using Unity's Universal Render Pipeline (URP) [S1, unofficial]. The method builds on the concept of triangle projection voxelization, where triangles are rendered into a voxel grid [5432-triangle-projection-voxelization.md]. The specific approach of choosing the projection axis based on the face normal is a heuristic to optimize the rasterization process for voxel coverage [S1, unofficial].

## Sources
[S1] 【Unity】GPUで3Dモデルをリアルタイムにボクセル化する, https://qiita.com/seyakara/items/b635939f6c6106a15d40, fetched 2026-10-29 (unofficial)
