# Voxel Face Mesh Generation

Also called: Block Face Culling, Visible Block Faces Meshing, Voxel Quad Generation

## Overview
Voxel Face Mesh Generation is a surface extraction method that produces a single quad mesh for every exposed voxel face in a discrete volume. It solves the problem of converting a blocky, discrete representation into a renderable triangle mesh by emitting one unit quad per visible face, leveraging 1-voxel padding to avoid boundary checks during neighbor lookups [S1, S2, unofficial]. Unlike smooth isosurface methods, it preserves the exact axis-aligned geometry of the voxel grid, and unlike greedy meshing, it does not merge adjacent coplanar quads into larger rectangles, resulting in a higher primitive count but simpler per-face logic [S1, S2, unofficial].

## How it works
The method operates on a padded 3D grid (chunk) where each cell holds a block type or material ID. It iterates through interior voxels, checks all six adjacent neighbors, and generates geometry only for faces where the neighbor is not "empty" (or meets specific visibility criteria). The output is a flat mesh data structure containing vertices, indices, normals, UVs, and tangents.

### Inputs and Outputs
*   **Input**: A 3D array of block types (or occupancy states) of size $N \times N \times N$, padded with a 1-voxel border of empty space.
*   **Output**: A `MeshData` structure containing dynamic arrays for vertices, indices, normals, UVs, tangents, and vertex colors, populated with one quad (2 triangles, 4 vertices) per exposed face.

### Data Structures
The implementation relies on static lookup tables and dynamic buffers [S1, S2, unofficial].

**MeshData Layout** [S1, unofficial]:
*   `vertices`: `Array<FVector>`
*   `triangles`: `Array<int>`
*   `normals`: `Array<FVector>`
*   `uv0`: `Array<FVector2D>`
*   `tangents`: `Array<FProcMeshTangent>`
*   `vertexColors`: `Array<FLinearColor>`
*   `VERTICES[6][4]`: Static base quad vertices for each of the 6 face directions.
*   `NORMALS[6]`: Static normal vectors for each face direction.
*   `UVS_Inverted[6][4]`: Static inverted UV coordinates for each face direction.
*   `TANGENTS[6]`: Static tangent vectors for each face direction.
*   `BlockTypeTextureIndex[MAX_TYPES][6]`: Lookup table mapping block type and face direction to an atlas index.
*   `ATLAS_SIZE`: Float constant defining the resolution of the texture atlas.

**Quad Structure** [S2, unofficial]:
*   `voxel`: `[x, y, z]` coordinates of the source voxel.
*   `width`: 1.0
*   `height`: 1.0

### Parameters
*   `BlockSize`: The world unit size of a single voxel, used to scale vertex positions [S1, unofficial].
*   `ATLAS_SIZE`: The resolution of the texture atlas (e.g., 16, 32), used to compute UV scaling [S1, unofficial].
*   `flip_v`: Boolean flag; `true` for side faces (XNeg, XPos, ZNeg, ZPos), `false` for top/bottom faces [S2, unofficial].

### Numbered Steps

1.  **Initialize Buffers**: Clear `MeshData` arrays. Set `lastNumVertices` to 0.
2.  **Iterate Voxels**: Loop through interior voxel indices $i, j, k$ from $1$ to $PADDED\_CHUNK\_MAX\_INDEX - 1$ (exclusive). This range excludes the 1-voxel padding border [S2, unofficial].
3.  **Check Face Visibility**: For each voxel at $(i, j, k)$, iterate through all 6 face directions $d \in \{UP, DOWN, LEFT, RIGHT, FORWARD, BACK\}$.
    *   Calculate neighbor coordinates: $(i + n_x, j + n_y, k + n_z)$, where $(n_x, n_y, n_z)$ is the normal for direction $d$.
    *   Linearize neighbor index: $idx_{neighbor} = \text{linearize}(i + n_x, j + n_y, k + n_z)$ [S2, unofficial].
    *   Determine visibility using the rule [S2, unofficial]:
        $$
        \text{face\_needs\_mesh}(V, N) = 
        \begin{cases} 
        \text{true} & \text{if } (V.\text{isOpaque} \land N.\text{isNotEmpty}) \lor (V.\text{isTranslucent} \land N.\text{isEmpty}) \\
        \text{false} & \text{otherwise}
        \end{cases}
        $$
        *Note: If the voxel is empty, no face is generated. If the neighbor is opaque, the face is hidden (culled) unless the current voxel is translucent [S2, unofficial].*
4.  **Generate Geometry**: If `face_needs_mesh` is true:
    *   **Vertices**: For each of the 4 base vertices $v \in \{0,1,2,3\}$ in `VERTICES[d]`:
        $$
        P_v = (V.\text{VERTICES}[d][v] \times 100) + (k \cdot \text{BlockSize}, j \cdot \text{BlockSize}, i \cdot \text{BlockSize})
        $$
        Add $P_v$ to `MeshData.vertices` [S1, unofficial].
    *   **Indices**: Add 6 indices to `MeshData.triangles` forming two triangles with consistent winding (counter-clockwise):
        $$
        \text{Indices} = [\text{lastNumVertices}+0, \text{lastNumVertices}+1, \text{lastNumVertices}+2, \text{lastNumVertices}+3, \text{lastNumVertices}+2, \text{lastNumVertices}+1]
        $$
        *Note: The second triangle uses winding (3,2,1) to maintain consistent face culling orientation [S1, unofficial].*
    *   **Normals**: Add 4 copies of `MeshData.NORMALS[d]` to `MeshData.normals` [S1, unofficial].
    *   **Tangents**: Add 4 copies of `MeshData.TANGENTS[d]` to `MeshData.tangents` [S1, unofficial].
    *   **Vertex Colors**: Add 4 copies of `FLinearColor(0.75, 0.75, 0.75, 1.0)` to `MeshData.vertexColors` [S1, unofficial].
    *   **UVs**:
        1.  Determine atlas index $I$ from `BlockTypeTextureIndex[blockType][d]` [S1, unofficial].
        2.  Compute atlas coordinates:
            $$
            UVI = I / \text{ATLAS\_SIZE}, \quad UVJ = I \% \text{ATLAS\_SIZE}
            $$
        3.  Compute UV size: $UVSize = 1.0 / \text{ATLAS\_SIZE}$ [S1, unofficial].
        4.  For each of the 4 base UVs $u \in \{0,1,2,3\}$ in `UVS_Inverted[d]`:
            $$
            UV_v = \text{FVector2D}(UVJ \cdot UVSize, UVI \cdot UVSize) + (u_x \cdot UVSize, u_y \cdot UVSize)
            $$
            *Note: Base UVs are inverted and components are swapped to align with the atlas layout [S1, unofficial].*
        5.  Add $UV_v$ to `MeshData.uv0` [S1, unofficial].

### Edge Cases and Pitfalls
*   **Winding Order**: The second triangle of the quad must use indices (3,2,1) instead of (0,1,2) to ensure consistent backface culling orientation across all face directions [S1, unofficial].
*   **UV Inversion**: UVs must be inverted and U/V components swapped using `FVector2D(UVJ * UVSize, UVI * UVSize)` to correctly map the atlas grid [S1, unofficial].
*   **Texture Fallback**: For specific block types (e.g., grass/snow), if the voxel directly above is opaque, the side face should use the `Face::Bottom` texture instead of the side texture [S2, unofficial].
*   **Padding**: Iteration must exclude the padded boundary to safely access neighbors without bounds checking [S2, unofficial].

## Subtopics

### [Texture Atlas UV Mapping](4238-texture-atlas-uv-mapping.md)
Maps block types to sub-regions of a shared texture atlas using a per-face lookup table, with UV inversion for correct winding order.

### [Face Direction and Quad Geometry](4881-face-direction-and-quad-geometry.md)
Defines the six axis-aligned face directions on a cube and provides helpers for generating quad geometry (positions, indices, UVs) for meshing.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel is visited once and up to 6 faces are checked.
*   **Memory Complexity**: $O(F)$ where $F$ is the number of exposed faces. In dense volumes, $F \approx 2N$, leading to a mesh size of roughly $2N$ quads.
*   **Performance**: The method is simple and cache-friendly due to the linear iteration and padding. However, the high primitive count (quads instead of merged quads) can lead to higher draw call overhead if not batched.
*   **Strengths**: Simple implementation, no complex merging logic, exact voxel alignment.
*   **Limitations**: High vertex/face count compared to greedy meshing. Not suitable for highly optimized rendering of large volumes without further optimization (e.g., frustum culling, occlusion culling).
*   **When to Use**: When exact voxel alignment is required and performance is not the primary bottleneck, or as a baseline for more complex meshing techniques.
*   **When Not to Use**: When rendering very large volumes with limited GPU bandwidth, where greedy meshing or octree-based approaches would reduce primitive count significantly.

## Background
Voxel Face Mesh Generation is a fundamental technique in voxel-based game engines and 3D modeling tools. It is widely used in games like Minecraft and its clones to render blocky worlds. The method relies on the concept of "face culling" to remove internal faces, a principle shared with other blocky meshing methods [S1, S2, unofficial]. The specific implementation details, such as the use of 1-voxel padding and specific UV mapping strategies, are derived from practical implementations in game engines like Unreal Engine [S1, unofficial] and Rust-based voxel engines [S2, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
[S2] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
