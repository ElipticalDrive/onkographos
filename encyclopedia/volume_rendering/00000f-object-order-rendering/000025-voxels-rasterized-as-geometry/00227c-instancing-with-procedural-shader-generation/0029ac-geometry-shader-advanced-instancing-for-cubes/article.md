# Geometry Shader Advanced Instancing for Cubes
Also called: Frustum Culling in Projection Space, GS Cube Instancing

## Overview
A technique using geometry shaders to generate cube meshes from instanced points, optimizing for visibility by emitting at most 12 vertices (3 quads) per cube [S1, unofficial]. It differs from creating full 24-vertex cubes by leveraging the fact that backface culling always removes one of two opposite faces, allowing the shader to emit only the visible subset [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Instanced voxel center coordinates $(x, y, z)$, a precomputed world-view-projection matrix $M_{WVP}$, a precomputed inverse projection direction vector $v_{InvProj}$, and a voxel code mask (e.g., `vs_out_VoxelCode`) indicating internal/external faces [S1, unofficial].
*   **Outputs:** Geometry shader emits up to 3 quads (12 vertices) per input point [S1, unofficial].

**Data Structures:**
*   **Corner Vectors:** Precomputed 3D vectors representing the corners of a unit cube relative to its center, added in projection space. Let $c_{vCornerijk}$ be these precomputed vectors where $i,j,k \in \{0,1\}$ [S1, unofficial].
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

3.  **Backface Culling in View Space:** Determine the visible face orientation. Transform a direction from projection space to view space using the inverse projection diagonal:
    $$v_{InvProj} = \left( \frac{1}{M_{Proj}[0][0]}, \frac{1}{M_{Proj}[1][1]}, \frac{1}{M_{Proj}[2][2]} \right)$$
    [S1, unofficial]

4.  **Face Visibility Check:** Compute the dot product between the transformed corner direction and the view direction $v_{View}$ to decide between front and back faces. Reject faces marked as "inside" the model using the voxel code mask [S1, unofficial].

5.  **Emit Geometry:** Emit vertices for the visible quads by adding the precomputed corner vectors to $v_{Pos}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed constants
c_vCorner000 = vec3(0.0, 0.0, 0.0) # Example placeholder; actual values depend on cube orientation
c_vCorner100 = vec3(1.0, 0.0, 0.0)
c_vCorner010 = vec3(0.0, 1.0, 0.0)
c_vCorner110 = vec3(1.0, 1.0, 0.0)
# Note: In the source, corners are precomputed in projection space. 
# The following assumes c_vCorner vectors are already in projection space.

# Input: x, y, z (voxel center), vs_out_VoxelCode (uint)
# Uniforms: c_mWorldViewProjection (mat4), c_mProjection (mat4), vViewDirection (vec3)

# Step 1: Transform to Clip Space
vPos = vec4(x, y, z, 1.0) * c_mWorldViewProjection

# Step 2: Frustum Culling
# Calculate expanded w to include corner extents
max_corner_len = max(
    max(length(c_vCorner000), length(c_vCorner001)),
    max(length(c_vCorner010), length(c_vCorner011))
)
w = vPos.w + max_corner_len

# Discard if outside expanded frustum
if abs(vPos.z) > w or abs(vPos.x) > w or abs(vPos.y) > w:
    return

# Step 3 & 4: Backface Culling and Face Masking
# Transform projection space direction to view space
inv_proj_xyz = vec3(
    1.0 / c_mProjection[0][0],
    1.0 / c_mProjection[1][1],
    1.0 / c_mProjection[2][2]
)

# Check front face
# Note: Source uses (c_vCorner000.xyz + c_vCorner110.xyz) as a direction vector
front_face_dir = (c_vCorner000.xyz + c_vCorner110.xyz) * inv_proj_xyz
if dot(front_face_dir, vViewDirection) > 0.0:
    # Check if front face is visible (not internal)
    if (vs_out_VoxelCode[0] & 0x10) != 0:
        # Emit Quad 0 (Front Face)
        # Vertices: 000, 100, 010, 110
        gl_Position = c_vCorner000 + vPos
        EmitVertex()
        gl_Position = c_vCorner100 + vPos
        EmitVertex()
        gl_Position = c_vCorner010 + vPos
        EmitVertex()
        gl_Position = c_vCorner110 + vPos
        EmitVertex()
        EndPrimitive()
else:
    # Check back face
    # Note: Source implies checking back face if front is not visible
    # Check if back face is visible (not internal)
    if (vs_out_VoxelCode[0] & 0x20) != 0:
        # Emit Quad 1 (Back Face)
        # Vertices: 001, 101, 011, 111 (Assuming standard cube indexing)
        # Source code snippet cuts off, but logic implies emitting the opposite face
        # Implementation detail: Use corresponding precomputed back corners
        pass # Implementation depends on specific corner precomputation

# Note: The source mentions emitting at most 12 vertices (3 quads). 
# The provided snippet only shows logic for one pair of opposite faces.
# A complete implementation would repeat the visibility check for the other two axis-aligned pairs.
```

**Parameters:**
*   **Frustum Expansion:** The offset added to $w$ is the maximum length of the precomputed corner vectors [S1, unofficial].
*   **Mask Bits:** `0x10` and `0x20` are used to distinguish internal vs. external faces for specific axis pairs [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Frustum Size:** Testing only the center causes voxels to disappear too early. An offset is added to "make the frustum larger" [S1, unofficial].
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
