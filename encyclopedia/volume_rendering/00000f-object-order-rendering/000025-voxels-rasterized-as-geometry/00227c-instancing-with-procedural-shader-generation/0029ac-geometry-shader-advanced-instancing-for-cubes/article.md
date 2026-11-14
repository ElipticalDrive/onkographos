# Geometry Shader Advanced Instancing for Cubes
Also called: Frustum Culling in Projection Space, GS Cube Instancing

## Overview
A technique using geometry shaders to generate cube meshes from instanced points, optimizing for visibility by emitting at most 12 vertices (3 quads) per cube [S1, unofficial]. It differs from creating full 24-vertex cubes by leveraging the fact that backface culling always removes one of two opposite faces, allowing the shader to emit only the visible subset [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Instanced voxel center coordinates $(x, y, z)$, a precomputed world-view-projection matrix $M_{WVP}$, a precomputed inverse projection direction vector $v_{InvProj}$, and a voxel code mask (e.g., `vs_out_VoxelCode`) indicating internal/external faces [S1, unofficial].
*   **Outputs:** Geometry shader emits up to 3 quads (12 vertices) per input point [S1, unofficial].

**Data Structures:**
*   **Corner Vectors:** Precomputed 3D vectors representing the corners of a unit cube relative to its center, added in projection space [S1, unofficial]. Let $c_{vCornerijk}$ be these precomputed vectors where $i,j,k \in \{0,1\}$ [S1, unofficial].
*   **Voxel Code:** A bitmask where specific bits indicate face visibility (e.g., bit 4 for one face, bit 5 for the opposite) [S1, unofficial].

**Steps:**
1.  **Transform to Clip Space:** Compute the homogeneous position of the voxel center in clip space:
    $$v_{Pos} = \begin{bmatrix} x & y & z & 1 \end{bmatrix} \times M_{WVP}$$
    [S1, unofficial]

2.  **Frustum Culling in Projection Space:** Calculate an expanded width $w$ to prevent premature culling of voxels near the frustum boundary:
    $$w = v_{Pos}.w + \max(\max(\|c_{vCorner000}\|, \|c_{vCorner001}\|), \max(\|c_{vCorner010}\|, \|c_{vCorner011}\|))$$
    Discard the voxel if it is outside the expanded viewing volume:
    $$\text{if } |v_{Pos}.z| > w \lor |v_{Pos}.x| > w \lor |v_{Pos}.y| > w \text{ then return}$$
    [S1, unofficial]

3.  **Backface Culling in View Space:** Determine the visible face orientation by transforming a direction from projection space to view space using the inverse projection diagonal:
    $$v_{InvProj} = \left( \frac{1}{M_{Proj}[0][0]}, \frac{1}{M_{Proj}[1][1]}, \frac{1}{M_{Proj}[2][2]} \right)$$
    [S1, unofficial]

4.  **Face Visibility Check:** Compute the dot product between the transformed corner direction and the view direction $v_{View}$ to decide between front and back faces [S1, unofficial]. Reject faces marked as "inside" the model using the voxel code mask [S1, unofficial].

5.  **Emit Geometry:** Emit vertices for the visible quads by adding the precomputed corner vectors to $v_{Pos}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed constants (provided by application or vertex shader)
# c_vCorner000, c_vCorner001, c_vCorner010, c_vCorner011,
# c_vCorner100, c_vCorner101, c_vCorner110, c_vCorner111
# All are vec3 in projection space.

def geometry_shader_advanced_instancing_cubes(x, y, z, M_WVP, M_Proj, v_view_dir, voxel_code):
    vertices = []

    # Step 1: Transform to Clip Space
    vPos = vec4(x, y, z, 1.0) @ M_WVP

    # Step 2: Frustum Culling in Projection Space
    max_corner_len = max(max(length(c_vCorner000), length(c_vCorner001)), 
                         max(length(c_vCorner010), length(c_vCorner011)))
    w = vPos.w + max_corner_len
    
    if abs(vPos.z) > w or abs(vPos.x) > w or abs(vPos.y) > w:
        return vertices

    # Step 3: Backface Culling in View Space
    inv_proj = vec3(1.0 / M_Proj[0][0], 1.0 / M_Proj[1][1], 1.0 / M_Proj[2][2])

    # Step 4 & 5: Face Visibility Check and Emit Geometry (Z-axis pair)
    # Direction vector for the face defined by corners 000 and 110
    face_dir = (c_vCorner000.xyz + c_vCorner110.xyz) * inv_proj
    
    if dot(face_dir, v_view_dir) > 0.0:
        # Front face visible
        if (voxel_code & 0x10) != 0:
            vertices.append(c_vCorner000 + vPos)
            vertices.append(c_vCorner100 + vPos)
            vertices.append(c_vCorner010 + vPos)
            vertices.append(c_vCorner110 + vPos)
    else:
        # Back face visible
        if (voxel_code & 0x20) != 0:
            vertices.append(c_vCorner001 + vPos)
            vertices.append(c_vCorner101 + vPos)
            vertices.append(c_vCorner011 + vPos)
            vertices.append(c_vCorner111 + vPos)

    # The source provides logic for one axis pair.
    # The other two axis pairs (X and Y) follow the identical structure
    # using their respective corner vectors and mask bits.
    return vertices
```

**Parameters:**
*   **Frustum Expansion:** The offset added to $w$ is the maximum length of the precomputed corner vectors [S1, unofficial].
*   **Mask Bits:** `0x10` and `0x20` are used to distinguish internal vs. external faces for specific axis pairs [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Frustum Size:** Testing only the center causes voxels to disappear too early [S1, unofficial]. An offset is added to "make the frustum larger" [S1, unofficial].
*   **Projection Space:** Backface culling cannot be done directly in projection space; it requires transformation to view space using the inverse projection diagonal [S1, unofficial].
*   **Internal Faces:** Neighboring voxels share faces; these must be rejected using a mask to avoid rendering invisible internal geometry [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel for culling and geometry generation, as the number of emitted vertices is bounded (max 12) [S1, unofficial].
*   **Memory Complexity:** Low, as geometry is generated procedurally in the shader without storing full mesh data [S1, unofficial].
*   **Strengths:** Significant reduction in vertex count compared to full cube rendering (12 vs 24 vertices) due to backface culling [S1, unofficial]. Efficient frustum culling in projection space [S1, unofficial].
*   **Limitations:** Requires precomputed corner vectors in projection space [S1, unofficial]. Backface culling logic is complex due to projection space transformations [S1, unofficial].
*   **When to Use:** When rendering large volumes of cubes where visibility culling is critical and hardware instancing is available [S1, unofficial].

## Background
Originated by Johannes Jendersie in a blog post on his developer blog [S1, unofficial]. The technique was developed to optimize voxel-based model rendering by reducing the vertex load on the GPU through selective geometry generation [S1, unofficial].

## Sources
[S1] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
