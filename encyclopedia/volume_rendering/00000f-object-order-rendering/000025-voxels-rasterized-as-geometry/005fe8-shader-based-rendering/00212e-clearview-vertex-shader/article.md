# ClearView Vertex Shader

Also called: HexMeshUnified Vertex

## Overview
The ClearView Vertex Shader reconstructs face geometry and attributes from texture buffers using `gl_VertexID`, avoiding vertex attribute buffers [S1, unofficial]. It processes 54,000 per-cell faces, each with 4 vertices and 2 triangles (6 indices) [S1, unofficial]. It differs from [Voxel Fragment Shader Rendering](5849-voxel-fragment-shader-rendering.md) by deferring attribute lookup to the vertex stage and relying on explicit face indexing rather than implicit voxel topology [G].

## How it works
**Inputs and Outputs:**
The shader takes no vertex attributes. It reads from three texture buffers: `uFaceTex` (face connectivity), `uVertexTex` (vertex positions/attributes), and `uEdgeTex` (edge attributes/LOD) [S1, unofficial]. It outputs `gl_Position` and varying attributes for the fragment shader [G].

**Data Structures:**
1.  **Face Buffer (`uFaceTex`)**: `RG32UI` texture with width $W_{face} = \text{uTexWidth}$ and height $H_{face} = \lceil N_{faces} \times 2 / W_{face} \rceil$ [S1, unofficial]. Each face is stored in two consecutive texels (linear index $2i$ and $2i+1$) [S1, unofficial].
    *   Texel $2i$: Vertex indices $vIdx[0..3]$ packed in RGBA [S1, unofficial].
    *   Texel $2i+1$: Edge indices $eIdx[0..3]$ packed in RGBA [S1, unofficial].
2.  **Vertex Buffer (`uVertexTex`)**: `RGBA32F` texture with width $W_{vert} = \text{uTexWidth}$ and height $H_{vert} = \lceil N_{verts} / W_{vert} \rceil$ [S1, unofficial]. Each texel stores position $(x,y,z)$ in RGB and scalar attribute $a$ in A [S1, unofficial].
3.  **Edge Buffer (`uEdgeTex`)**: `RG32F` texture with width $W_{edge} = \text{uTexWidth}$ and height $H_{edge} = \lceil N_{edges} / W_{edge} \rceil$ [S1, unofficial]. Each texel stores line attribute $r$ and LOD value $g$ in RG [S1, unofficial].

**Steps:**
1.  **Calculate Face ID**: Compute $F = \lfloor \text{gl\_VertexID} / 6 \rfloor$ [S1, unofficial].
2.  **Fetch Face Data**: Fetch `faceVertices` from `uFaceTex` at linear index $2F$. Extract $vIdx[0..3]$ from RGBA. Fetch `faceEdges` from `uFaceTex` at linear index $2F+1$. Extract $eIdx[0..3]$ from RGBA [S1, unofficial].
3.  **Fetch Vertex/Edge Attributes**: Initialize arrays `vVertexPositions[4]`, `vVertexAttributes[4]`, `vLineAttributes[4]`, `vEdgeLodValues[4]`. For $i = 0$ to $3$: Fetch `vData` from `uVertexTex` at index $vIdx[i]$. Set `vVertexPositions[i] = vData.xyz`, `vVertexAttributes[i] = vData.a`. Fetch `eData` from `uEdgeTex` at index $eIdx[i]$. Set `vLineAttributes[i] = eData.r`, `vEdgeLodValues[i] = eData.g` [S1, unofficial].
4.  **Determine Corner ID**: Map `gl_VertexID % 6` to a corner index $C$ for the two triangles of the quad face: Vertex 0 $\rightarrow$ Corner 0, Vertex 1 $\rightarrow$ Corner 3, Vertex 2 $\rightarrow$ Corner 1, Vertex 3 $\rightarrow$ Corner 2, Vertex 4 $\rightarrow$ Corner 1, Vertex 5 $\rightarrow$ Corner 3 [S1, unofficial].
5.  **Transform and Output**: Fetch `thisVertexData` from `uVertexTex` at index $vIdx[C]$. Compute world position: $P_{world} = M_{model} \times \text{vec4}(thisVertexData.xyz, 1.0)$. Set `vFragmentPositionWorld = P_world.xyz`. Set `gl_Position = M_{mvp} \times \text{vec4}(thisVertexData.xyz, 1.0)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# ClearView Vertex Shader Pseudocode
def clearview_vertex_shader(gl_VertexID, uTexWidth, uFaceTex, uVertexTex, uEdgeTex, uModelMatrix, uModelViewProjection):
    # 1. Calculate Face ID
    face_id = gl_VertexID // 6

    # 2. Fetch Face Data (linearToTexel maps linear index to 2D texel coordinates)
    face_vertices = uFaceTex[face_id * 2]
    face_edges = uFaceTex[face_id * 2 + 1]
    v_idx = face_vertices.rgba
    e_idx = face_edges.rgba

    # 3. Fetch Vertex/Edge Attributes
    v_vertex_positions = [None] * 4
    v_vertex_attributes = [None] * 4
    v_line_attributes = [None] * 4
    v_edge_lod_values = [None] * 4

    for i in range(4):
        v_data = uVertexTex[v_idx[i]]
        v_vertex_positions[i] = v_data.xyz
        v_vertex_attributes[i] = v_data.a

        e_data = uEdgeTex[e_idx[i]]
        v_line_attributes[i] = e_data.r
        v_edge_lod_values[i] = e_data.g

    # 4. Determine Corner ID
    corner_id_map = [0, 3, 1, 2, 1, 3]
    corner_id = corner_id_map[gl_VertexID % 6]

    # 5. Transform and Output
    current_v_idx = v_idx[corner_id]
    current_v_data = uVertexTex[current_v_idx]
    world_pos = uModelMatrix * vec4(current_v_data.xyz, 1.0)

    vFragmentPositionWorld = world_pos.xyz
    gl_Position = uModelViewProjection * vec4(current_v_data.xyz, 1.0)
```

**Parameters and Edge Cases:**
*   `uModelMatrix`, `uModelViewProjection`: Standard transformation matrices [G].
*   `uTexWidth`: Texture width used for linear indexing [S1, unofficial].
*   **Precision**: Linear indexing assumes integer arithmetic. Ensure `uTexWidth` is large enough to avoid overflow for large meshes [G].
*   **Boundaries**: `gl_VertexID` must be within $[0, 6 \times N_{faces})$ [G].

## Subtopics

### HexMeshUnified Vertex
A variant optimized for hexahedral meshes, sharing the same reconstruction principle but with specific topology assumptions. [HexMeshUnified Vertex](5850-hexmeshunified-vertex.md)

### ClearView Fragment Shader
Handles the per-fragment rendering, including line drawing, LOD-based opacity, and alpha blending. [ClearView Fragment Shader](5851-clearview-fragment-shader.md)

## Operating characteristics
*   **Time Complexity**: $O(1)$ per vertex, involving multiple texture fetches (2 face, 4 vertex/edge, 1 current vertex) [G].
*   **Memory Complexity**: $O(N_{faces} + N_{verts} + N_{edges})$ in texture memory. No vertex buffer object (VBO) memory required for attributes [G].
*   **Strengths**: Reduces VRAM usage by removing vertex attributes from VBOs. Flexible topology [G].
*   **Limitations**: High texture fetch overhead. Cache misses on `uVertexTex` and `uEdgeTex` can degrade performance. Not suitable for extremely high-frequency vertex access patterns without careful texture tiling [G].
*   **When to use**: Sparse or dynamic meshes where vertex attributes change frequently or memory is constrained [G].
*   **When not to use**: Static meshes with simple topology where VBOs are more efficient [G].
*   **Measured Performance**: Not given in the sources.

## Background
The ClearView Vertex Shader is part of the ClearView rendering system, developed for interactive visualization of large-scale volumetric and mesh data [G]. It was introduced in the context of the HexVolumeRenderer project, aiming to balance visual fidelity with performance on consumer hardware [S1, unofficial]. The method avoids traditional vertex buffer overhead by encoding geometry in texture buffers [S1, unofficial].

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
