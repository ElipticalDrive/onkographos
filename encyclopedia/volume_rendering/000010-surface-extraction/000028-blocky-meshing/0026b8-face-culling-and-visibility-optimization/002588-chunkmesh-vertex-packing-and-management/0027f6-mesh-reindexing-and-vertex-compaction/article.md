# Mesh Reindexing and Vertex Compaction

Also called: Mesh Simplification via Vertex Deduplication, Vertex Compaction, compactVertices

## Overview
This method reduces the memory footprint of a chunk-based mesh by removing unreferenced vertices and renumbering the remaining ones to form a dense, contiguous array. It solves the problem of sparse vertex buffers that arise during incremental blocky meshing, where vertices are allocated per voxel face but many are culled or unused. Unlike [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md), which focuses on storage layout, this method actively compacts the data. It differs from [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md) by performing the logical remapping rather than just updating buffer pointers.

## How it works
**Inputs:** A mesh object containing a position buffer `mPositions`, an index collection `mIndexCollection`, and optional attribute buffers (`mNormals`, `mTexCoords`, and generic `mMesh->getAttributes()`).
**Outputs:** The same mesh object with compacted buffers and remapped indices.

**Data Structures:**
*   `mPositions`: Array of `Position` structs. Size $N$.
*   `mIndexCollection`: Collection of indices (e.g., `std::vector<Index>` or similar). Each index has a component `.x()` representing the vertex ID.
*   `mNormals`: Array of `Normal` structs. Size $N$ (if present).
*   `mTexCoords`: Array of `TexCoord` structs. Size $N$ (if present).
*   `mMesh->getAttributes()`: List of generic attributes. Each has `getSize()`, `getNumComponents()`, `getFormat()`, and `assign()` method.

**Steps:**
1.  **Implicit Index Check:** If the mesh uses implicit indices (e.g., position-based indexing where vertex ID equals array index), return immediately [S1, unofficial].
2.  **Mark Used Vertices:** Create a boolean vector `used` of size $N$, initialized to `false`. Iterate through `mIndexCollection`. For each index `idx`, if `idx.x()` is within bounds $[0, N)$, set `used[idx.x()] = true` [S1, unofficial].
3.  **Build Remap Table:** Create a remap table `remap` of size $N$, initialized to `std::size_t(-1)`. Iterate `oldIndex` from $0$ to $N-1$. If `used[oldIndex]` is true, assign `remap[oldIndex] = newIndex` and increment `newIndex`. The final value of `newIndex` is the new compacted vertex count $N'$ [S1, unofficial].
4.  **Compact Positions:** Create `newPositions` of size $N'$. Iterate `oldIndex` from $0$ to $N-1$. If `remap[oldIndex] != std::size_t(-1)`, set `newPositions[remap[oldIndex]] = mPositions[oldIndex]`. Update `mPositions` with `newPositions` [S1, unofficial].
5.  **Compact Normals:** If `mNormals` exists and `mNormals.getSize() == N`, create `newNormals` of size $N'$. Iterate `oldIndex` from $0$ to $N-1$. If `remap[oldIndex] != std::size_t(-1)`, set `newNormals[remap[oldIndex]] = mNormals[oldIndex]`. Update `mNormals` [S1, unofficial].
6.  **Compact TexCoords:** If `mTexCoords` exists and `mTexCoords.getSize() == N`, create `newTexCoords` of size $N'$. Iterate `oldIndex` from $0$ to $N-1$. If `remap[oldIndex] != std::size_t(-1)`, set `newTexCoords[remap[oldIndex]] = mTexCoords[oldIndex]`. Update `mTexCoords` [S1, unofficial].
7.  **Compact Generic Attributes:** For each attribute `attr` in `mMesh->getAttributes()`:
    *   If `attr.getSize() == N`:
        *   Create `newAttr` with size $N'$, same component count and format as `attr`.
        *   Iterate `oldIndex` from $0$ to $N-1$. If `remap[oldIndex] != std::size_t(-1)`, call `newAttr.assign(remap[oldIndex], oldIndex, attr)` to copy the data [S1, unofficial].
        *   Replace `attr` with `newAttr` [S1, unofficial].
    *   If `attr.getSize() != N`, skip it to prevent corruption [S1, unofficial].
8.  **Remap Indices:** Iterate through `mIndexCollection`. For each `idx`, update `idx.x() = static_cast<std::uint32_t>(remap[idx.x()])` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compact_vertices(mesh):
    # Step 1: Check for implicit indices
    if mesh.has_implicit_indices():
        return

    n = mesh.m_positions.size()
    if n == 0:
        return

    # Step 2: Mark used vertices
    used = [False] * n
    for idx in mesh.m_index_collection:
        vi = idx.x()
        if 0 <= vi < n:
            used[vi] = True

    # Step 3: Build remap table
    remap = [-1] * n
    new_index = 0
    for old_index in range(n):
        if used[old_index]:
            remap[old_index] = new_index
            new_index += 1

    new_n = new_index

    # Step 4: Compact positions
    new_positions = [None] * new_n
    for old_index in range(n):
        if remap[old_index] != -1:
            new_positions[remap[old_index]] = mesh.m_positions[old_index]
    mesh.m_positions.set_values(new_positions)

    # Step 5: Compact normals
    if mesh.m_normals is not None and mesh.m_normals.size() == n:
        new_normals = [None] * new_n
        for old_index in range(n):
            if remap[old_index] != -1:
                new_normals[remap[old_index]] = mesh.m_normals[old_index]
        mesh.m_normals.set_values(new_normals)

    # Step 6: Compact texcoords
    if mesh.m_texcoords is not None and mesh.m_texcoords.size() == n:
        new_texcoords = [None] * new_n
        for old_index in range(n):
            if remap[old_index] != -1:
                new_texcoords[remap[old_index]] = mesh.m_texcoords[old_index]
        mesh.m_texcoords.set_values(new_texcoords)

    # Step 7: Compact generic attributes
    for attr in mesh.m_mesh.get_attributes():
        if attr.size() == n:
            new_attr = RawArray()
            new_attr.set_num_components(attr.get_num_components())
            new_attr.set_format(attr.get_format())
            new_attr.set_size(new_n)
            for old_index in range(n):
                if remap[old_index] != -1:
                    new_attr.assign(remap[old_index], old_index, attr)
            # Replace attribute in place (implementation dependent, conceptually)
            # In Python-like pseudocode, we assume attr is replaced by new_attr
            # The source uses 'attr = std::move(newAttr)' which implies the container updates the reference
            # Here we simulate the update:
            # Note: In a real C++ implementation, this modifies the object in the list
            # For pseudocode clarity, we assume the attribute object is updated
            attr = new_attr 

    # Step 8: Remap indices
    for idx in mesh.m_index_collection:
        idx.x = static_cast_uint32(remap[idx.x])
```

**Parameters:**
*   `epsDiagonal`, `epsDuplicate`: Not used in this method (used in critical point detection [S1, unofficial]).
*   Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Implicit Indices:** If the mesh uses implicit indices, the method returns immediately to avoid undefined behavior [S1, unofficial].
*   **Misaligned Attributes:** Attributes must have the same size as the position buffer to be compacted. If sizes differ, they are skipped to prevent data corruption [S1, unofficial].
*   **Index Bounds:** Index values must be within $[0, N)$ to be marked as used. Out-of-bounds indices are ignored [S1, unofficial].

## Subtopics

### [Voxel Face Meshing with Packed Attributes](4153-voxel-face-meshing-with-packed-attributes.md)
Focuses on storing vertex attributes in a packed format. Choose when memory layout is the primary concern, not vertex count reduction.

### [Chunk Mesh Data Structure & Buffer Update](4182-chunk-mesh-data-structure-buffer-update.md)
Handles buffer updates and data structure management. Choose when integrating this compaction into a larger mesh update pipeline.

### [Alpha/Translucent Block Rendering](4860-alpha-translucent-block-rendering.md)
Handles translucent rendering. This compaction method can be applied to the mesh data used by this renderer.

## Operating characteristics
*   **Time Complexity:** $O(N + I)$, where $N$ is the number of vertices and $I$ is the number of indices. The method iterates through indices once to mark used vertices, then through vertices to build the remap table and compact buffers, and finally through indices to remap them.
*   **Memory Complexity:** $O(N)$ for the `used` and `remap` arrays, plus $O(N')$ for the new compacted buffers, where $N' \le N$.
*   **Strengths:** Reduces memory usage significantly when many vertices are unused. Simple to implement.
*   **Limitations:** Does not remove duplicate vertices (only unreferenced ones). Requires a full pass over the mesh data.
*   **When to use:** After face culling and visibility optimization, before rendering, to reduce the vertex buffer size.
*   **When not to use:** If the mesh uses implicit indices or if attributes are not aligned with the position buffer.

## Background
This method is part of the scientific visualization toolkit [naeemsharif-ai/scientific-visualization](https://github.com/naeemsharif-ai/scientific-visualization) [S1, unofficial]. It is used in the context of blocky meshing for volume rendering, specifically after chunk meshing and face culling steps.

## Sources
[S1] naeemsharif-ai/scientific-visualization, https://github.com/naeemsharif-ai/scientific-visualization, fetched 2026-10-26 (unofficial)
