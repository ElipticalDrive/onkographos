# Material-Based Mesh Sectioning

Also called: GenerateTriangles, Mesh Splitting by Material

## Overview
Material-Based Mesh Sectioning splits a decoded mesh into separate arrays of triangles based on the material ID associated with each triangle's vertices [S1, unofficial]. It solves the problem of organizing geometry for multi-material rendering by ensuring that all triangles sharing the same material ID are contiguous in memory, facilitating efficient GPU draw calls or further merging operations. It differs from its sibling [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md) by focusing on material-based partitioning rather than layer-based (e.g., opaque/transparent) assembly, and from [Material ID Assignment & Caching](13805-material-id-assignment-caching.md) by handling the geometric distribution rather than the logical resource management of materials.

## How it works
**Inputs:** A decoded mesh structure containing vertex positions and an index buffer representing triangles.
**Outputs:** An array of mesh sections, where each section corresponds to a unique material ID and contains the indices or vertices of triangles belonging to that material.

**Data Structures:**
*   `MeshSection`: A container holding the triangle data for a specific material. The exact internal layout (e.g., interleaved vertices vs. index buffer) is implementation-dependent but must support iteration over triangles.
*   `sections`: A dynamic array (or vector) of `MeshSection` objects. The index of a section in this array corresponds directly to its `materialID`.

**Steps:**
1.  **Decode Input:** Retrieve the decoded mesh using `GetDecodedMesh`. This ensures indices are valid and vertices are accessible [S1, unofficial].
2.  **Sanity Check:** If the number of indices in the mesh is less than 3, return an empty `sections` array immediately [S1, unofficial].
3.  **Initialize Sections:** Create an empty list `sections` to store the material-based partitions [S1, unofficial].
4.  **Iterate Triangles:** Loop through the index buffer in steps of 3. Let the current index be $i$. The three indices for the triangle are $idx_0 = indices[i]$, $idx_1 = indices[i+1]$, and $idx_2 = indices[i+2]$ [S1, unofficial].
5.  **Retrieve Vertices:** Get the vertex data for $idx_0$, $idx_1$, and $idx_2$. Let their positions be $P_0, P_1, P_2$ and their material IDs be $M_0, M_1, M_2$ [S1, unofficial].
6.  **Apply Offset:** If the mesh has a global position offset $O$, update the positions: $P'_k = P_k + O$ for $k \in \{0, 1, 2\}$ [S1, unofficial].
7.  **Determine Material ID:**
    *   Start with $M = M_0$ [S1, unofficial].
    *   If $M_1 == M_0$ and $M_2 == M_0$, then $M = M_0$ [S1, unofficial].
    *   If $M_1 == M_2$ and $M_1 \neq M_0$, the source notes imply a preference for consistency or a specific rule: "If Vertex 1 and Vertex 2 have the same material ID, use that instead" [S1, unofficial]. This suggests $M = M_1$ if $M_1 == M_2$.
    *   *Note:* The source does not specify behavior if all three IDs are different. [G] In such a case, the triangle is typically discarded or assigned to the first vertex's material to maintain validity, but this is not explicitly defined in the source.
8.  **Ensure Section Existence:** If $M \geq$ `sections.size()`, resize `sections` to $M + 1$, initializing new elements as empty sections [S1, unofficial].
9.  **Add Triangle:** Append the triangle (defined by $P'_0, P'_1, P'_2$) to `sections[M]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def material_based_mesh_sectioning(decoded_mesh):
    # Step 1: Decode
    mesh = GetDecodedMesh(decoded_mesh)
    
    # Step 2: Sanity Check
    if len(mesh.indices) < 3:
        return []
    
    sections = []
    offset = mesh.position_offset # [G] Assumed field from decoded mesh
    
    # Step 4: Iterate Triangles
    for i in range(0, len(mesh.indices), 3):
        idx0 = mesh.indices[i]
        idx1 = mesh.indices[i+1]
        idx2 = mesh.indices[i+2]
        
        # Step 5: Retrieve Vertices
        v0 = mesh.vertices[idx0]
        v1 = mesh.vertices[idx1]
        v2 = mesh.vertices[idx2]
        
        # Step 6: Apply Offset
        p0 = v0.position + offset
        p1 = v1.position + offset
        p2 = v2.position + offset
        
        m0 = v0.material_id
        m1 = v1.material_id
        m2 = v2.material_id
        
        # Step 7: Determine Material ID
        # Rule: If V1 and V2 have the same ID, use that. Otherwise use V0.
        if m1 == m2:
            material_id = m1
        else:
            material_id = m0
            
        # Step 8: Ensure Section Existence
        if material_id >= len(sections):
            # Resize to material_id + 1 with zeroed/empty elements
            new_sections = [MeshSection() for _ in range(material_id + 1)]
            new_sections[:len(sections)] = sections
            sections = new_sections
            
        # Step 9: Add Triangle
        sections[material_id].add_triangle(p0, p1, p2)
        
    return sections
```

**Parameters:**
*   None explicitly defined. The method relies on the structure of the decoded mesh.

**Edge Cases and Pitfalls:**
*   **Mixed Materials:** If a triangle has three different material IDs, the source rule prioritizes $M_0$ (since $M_1 \neq M_2$). This may result in a triangle being assigned to a material that does not cover all its vertices, potentially causing visual artifacts if the material is not uniform across the triangle.
*   **Sparse Material IDs:** If material IDs are large and sparse, resizing the `sections` array to $M+1$ can lead to significant memory waste. [G] In practice, a hash map or sparse array might be used, but the source specifies a dense array resize.
*   **Index Bounds:** The loop assumes the index count is a multiple of 3. If not, the last 1 or 2 indices are ignored.

## Subtopics

### [Material ID Assignment & Caching](13805-material-id-assignment-caching.md)
Maps block-level material identifiers to unique, globally distinct terrain material IDs to facilitate mesh merging and GPU resource management. It differs by focusing on logical disambiguation and caching rather than geometric sectioning. Choose this when managing GPU resources for identical materials.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of triangles. Each triangle is processed once. Resizing the `sections` array is amortized $O(1)$ per new material ID, but worst-case $O(M)$ if many new IDs are encountered sequentially.
*   **Memory Complexity:** $O(N + M)$, where $M$ is the maximum material ID encountered. The output stores all triangle data, plus the overhead of the `sections` array.
*   **Strengths:** Simple to implement; ensures material-contiguous geometry; handles sparse material IDs via dynamic resizing.
*   **Limitations:** Inefficient for sparse material IDs (memory waste); does not handle mixed-material triangles gracefully (arbitrary assignment); no optimization for triangle strip or fan formats (assumes triangle list).
*   **When to Use:** When the mesh is already in triangle list format and material IDs are dense and small.
*   **When Not to Use:** When material IDs are sparse or large; when mixed-material triangles are common and require splitting.

## Background
This method is derived from the VoxelGenerator project by Jay2645 [S1, unofficial]. It is part of the mesh assembly pipeline in voxel-based rendering systems, where geometry is generated from block data and needs to be organized for rendering.

## Sources
[S1] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
