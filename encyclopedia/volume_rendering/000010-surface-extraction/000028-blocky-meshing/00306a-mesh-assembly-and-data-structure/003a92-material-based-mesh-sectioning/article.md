# Material-Based Mesh Sectioning

Also called: GenerateTriangles, Mesh Splitting by Material

## Overview
Material-Based Mesh Sectioning splits a decoded mesh into separate sections (arrays of triangles) based on the material ID associated with each triangle's vertices [S1, unofficial]. It serves as the bridge between raw geometry generation and material-aware rendering, ensuring that triangles sharing the same material are grouped for efficient batching. It differs from [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md) by operating on per-triangle material IDs rather than per-vertex block types, and from [Material ID Assignment & Caching](13805-material-id-assignment-caching.md) by performing the actual geometric partitioning rather than just ID mapping.

## How it works
**Inputs:** A decoded mesh object containing vertex positions and an index buffer.
**Outputs:** An array of mesh sections, where each section contains triangles belonging to a specific material ID.

**Data Structures:**
*   `MeshSection`: A container holding a list of triangle indices or vertices.
*   `sections`: A dynamic array (or vector) of `MeshSection` objects, indexed by material ID.

**Steps:**
1.  **Decode:** Retrieve the decoded mesh data using `GetDecodedMesh`. This provides access to raw vertex positions and the index buffer [S1, unofficial].
2.  **Sanity Check:** If the number of indices in the mesh is less than 3, return an empty array of mesh sections [S1, unofficial].
3.  **Initialize Sections:** Create an empty list `sections` to store the resulting material groups.
4.  **Iterate Triangles:** Loop through the index buffer in steps of 3. Let the current index triplet be $i_0, i_1, i_2$.
5.  **Retrieve Vertices:** For each index $i_k$, retrieve the corresponding vertex $v_k$ from the vertex buffer.
    *   Apply mesh offset to the position: $p_k = v_k.position + \text{meshOffset}$ [S1, unofficial].
    *   Retrieve the material ID associated with each vertex: $m_k = v_k.materialID$.
6.  **Determine Triangle Material ID ($M_{tri}$):**
    *   Start with $M_{tri} = m_0$ (the material ID of the first vertex) [S1, unofficial].
    *   If $m_1 == m_2$, update $M_{tri} = m_1$. This rule prioritizes consistency when vertices 1 and 2 share a material, implying a preference for the material of the "edge" or "face" defined by those vertices over the starting vertex in mixed-material scenarios [S1, unofficial].
    *   *Note:* If $m_0, m_1, m_2$ are all different, the triangle is assigned to $m_0$. If $m_0 \neq m_1$ but $m_1 == m_2$, it is assigned to $m_1$.
7.  **Expand Sections Array:** Ensure `sections` is large enough to hold an entry for $M_{tri}$.
    *   If $M_{tri} \ge \text{len}(sections)$, resize `sections` to size $M_{tri} + 1$, initializing new elements as empty sections [S1, unofficial].
8.  **Add Triangle:** Append the triangle (defined by vertices $p_0, p_1, p_2$ or their indices) to `sections[M_{tri}]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def material_based_mesh_sectioning(decoded_mesh):
    # Step 1: Decode
    vertices = decoded_mesh.get_vertices()
    indices = decoded_mesh.get_indices()
    mesh_offset = decoded_mesh.get_offset()

    # Step 2: Sanity Check
    if len(indices) < 3:
        return []

    sections = []

    # Step 4: Iterate Triangles
    for i in range(0, len(indices), 3):
        idx0 = indices[i]
        idx1 = indices[i + 1]
        idx2 = indices[i + 2]

        # Step 5: Retrieve Vertices
        v0 = vertices[idx0]
        v1 = vertices[idx1]
        v2 = vertices[idx2]

        # Apply offset to positions
        p0 = v0.position + mesh_offset
        p1 = v1.position + mesh_offset
        p2 = v2.position + mesh_offset

        # Get material IDs
        m0 = v0.material_id
        m1 = v1.material_id
        m2 = v2.material_id

        # Step 6: Determine Triangle Material ID
        tri_material_id = m0
        if m1 == m2:
            tri_material_id = m1

        # Step 7: Expand Sections Array
        if tri_material_id >= len(sections):
            # Resize to tri_material_id + 1 with zeroed/empty elements
            new_size = tri_material_id + 1
            sections.extend([MeshSection() for _ in range(len(sections), new_size)])

        # Step 8: Add Triangle
        sections[tri_material_id].add_triangle(p0, p1, p2)

    return sections
```

**Edge Cases and Pitfalls:**
*   **Mixed Materials:** The rule `if m1 == m2: use m1` creates a deterministic but potentially arbitrary assignment for triangles with three distinct material IDs. Implementers must ensure shaders handle these "mixed" triangles correctly or that the mesh generation phase avoids such triangles [S1, unofficial].
*   **Sparse Material IDs:** If material IDs are sparse (e.g., IDs 0 and 1000 exist, but 1-999 do not), the `sections` array will be large and mostly empty. This is a memory inefficiency. [Material ID Assignment & Caching](13805-material-id-assignment-caching.md) addresses this by remapping IDs to a dense range before sectioning.
*   **Index Bounds:** Ensure `idx0, idx1, idx2` are within the bounds of the vertex buffer.

## Subtopics

### [Material ID Assignment & Caching](13805-material-id-assignment-caching.md)
Maps block material IDs to unique terrain material IDs to enable mesh merging, creating materials on demand and caching results to avoid redundant GPU allocations. Use this when material IDs are sparse or when merging geometry across chunks is required.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of triangles. Each triangle is processed exactly once.
*   **Memory Complexity:** $O(N + M)$, where $M$ is the maximum material ID. The `sections` array grows to accommodate the highest material ID, even if many IDs are unused.
*   **Strengths:** Simple, deterministic, and efficient for dense material ID spaces.
*   **Limitations:** Inefficient for sparse material IDs due to array resizing. Does not handle material blending or transparency sorting within the sectioning step itself.
*   **When to Use:** When material IDs are dense and contiguous, or as a preprocessing step before further optimization.

## Background
This method is derived from the VoxelGenerator project by Jay2645, which implements a chunk-based mesh assembly pipeline for voxel engines [S1, unofficial]. It reflects common practices in procedural terrain rendering where geometry is generated per-chunk and then split for rendering optimization.

## Sources
[S1] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
