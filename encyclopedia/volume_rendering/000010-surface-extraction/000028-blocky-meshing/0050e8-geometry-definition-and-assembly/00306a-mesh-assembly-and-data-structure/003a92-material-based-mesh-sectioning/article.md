# Material-Based Mesh Sectioning

Also called: GenerateTriangles, Mesh Splitting by Material

## Overview
Material-Based Mesh Sectioning splits a decoded mesh into separate sections (arrays of triangles) based on the material ID associated with each triangle's vertices [S1, unofficial]. It solves the problem of organizing geometry for GPU submission by grouping triangles that share the same material, enabling efficient batching and rendering [S1, unofficial]. It differs from sibling methods like [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md) by focusing on the logical disambiguation of material references rather than the geometric assembly or sectioning of the mesh itself [S1, unofficial].

## How it works
**Inputs:** A decoded mesh structure containing vertex positions, indices, and per-vertex material IDs.
**Outputs:** An array of mesh sections, where each section contains the triangles belonging to a specific material ID.

**Data Structures:**
*   `MeshSection`: A container for a single material group.
    *   `material_id`: Integer identifier for the material.
    *   `triangles`: List of triangle definitions (three vertex indices or positions).
*   `MeshSections`: Dynamic array of `MeshSection` objects.

**Steps:**
1.  **Decode Input:** Retrieve the decoded mesh using `GetDecodedMesh`.
2.  **Sanity Check:** If the number of indices is less than 3, return an empty `MeshSections` array.
3.  **Initialize Sections:** Create an empty `MeshSections` array.
4.  **Iterate Triangles:** Loop through the decoded mesh indices in steps of 3. Let $i$ be the current index.
5.  **Retrieve Vertices:** Get the three vertex indices $v_0 = \text{indices}[i]$, $v_1 = \text{indices}[i+1]$, $v_2 = \text{indices}[i+2]$.
6.  **Get Positions:** Retrieve the 3D positions $P_0, P_1, P_2$ for these vertices. Add the mesh offset to each position: $P'_k = P_k + \text{mesh\_offset}$ for $k \in \{0, 1, 2\}$.
7.  **Determine Material ID:**
    *   Let $m_0, m_1, m_2$ be the material IDs of vertices $v_0, v_1, v_2$.
    *   If $m_0 = m_1 = m_2$, set $m_{tri} = m_0$.
    *   Else if $m_1 = m_2$, set $m_{tri} = m_1$.
    *   Else, set $m_{tri} = m_0$.
    *   *Note:* This logic prioritizes consistency, preferring the material of the second and third vertices if they match, otherwise defaulting to the first [S1, unofficial].
8.  **Ensure Section Existence:** If $m_{tri} \ge \text{len}(\text{MeshSections})$, resize the array to $m_{tri} + 1$ with zeroed elements.
9.  **Add Triangle:** Append the triangle defined by $(P'_0, P'_1, P'_2)$ to `MeshSections[m_tri].triangles`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def material_based_mesh_sectioning(decoded_mesh):
    indices = decoded_mesh.indices
    vertices = decoded_mesh.vertices
    mesh_offset = decoded_mesh.offset
    
    if len(indices) < 3:
        return []
    
    mesh_sections = []
    
    for i in range(0, len(indices), 3):
        v0_idx = indices[i]
        v1_idx = indices[i+1]
        v2_idx = indices[i+2]
        
        p0 = vertices[v0_idx] + mesh_offset
        p1 = vertices[v1_idx] + mesh_offset
        p2 = vertices[v2_idx] + mesh_offset
        
        m0 = vertices[v0_idx].material_id
        m1 = vertices[v1_idx].material_id
        m2 = vertices[v2_idx].material_id
        
        if m0 == m1 == m2:
            material_id = m0
        elif m1 == m2:
            material_id = m1
        else:
            material_id = m0
            
        if material_id >= len(mesh_sections):
            # Resize array to hold new section, initialized with empty triangles
            new_sections = [None] * (material_id + 1)
            for j in range(len(mesh_sections)):
                new_sections[j] = mesh_sections[j]
            mesh_sections = new_sections
            
        if mesh_sections[material_id] is None:
            mesh_sections[material_id] = {
                'material_id': material_id,
                'triangles': []
            }
            
        mesh_sections[material_id]['triangles'].append((p0, p1, p2))
        
    return mesh_sections
```

**Edge Cases and Pitfalls:**
*   **Mixed Materials:** If a triangle has vertices with different material IDs, the logic above assigns a single material ID based on the specified priority. This may result in visual artifacts if the material boundary cuts through the triangle.
*   **Sparse Material IDs:** If material IDs are large and sparse, resizing the array to `material_id + 1` can lead to significant memory waste. A hash map or dictionary might be more efficient in such cases, though the source specifies array resizing [S1, unofficial].
*   **Empty Sections:** The source mentions resizing with zeroed elements, implying empty sections are valid entries in the array.

## Subtopics

### [Material ID Assignment & Caching](13805-material-id-assignment-caching.md)
Maps block-level material identifiers to unique, globally distinct terrain material IDs to facilitate mesh merging and GPU resource management. It creates material objects on demand and caches the results to avoid redundant GPU allocations. It differs from sibling methods by focusing on the logical disambiguation and caching of material references rather than the geometric assembly or sectioning of the mesh itself.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of triangles, as each triangle is processed exactly once.
*   **Memory Complexity:** $O(N + M)$, where $M$ is the maximum material ID used. The array size depends on the highest material ID, not the number of unique materials.
*   **Strengths:** Simple implementation, efficient for dense material ID spaces, facilitates GPU batching.
*   **Limitations:** Inefficient for sparse material ID spaces; potential for visual artifacts if triangles span multiple materials.
*   **When to Use:** When material IDs are dense and small integers, and GPU batching by material is required.
*   **When Not to Use:** When material IDs are sparse or large, or when triangles frequently span multiple materials.

## Background
Originating from the VoxelGenerator project by Jay2645, this method is part of the mesh assembly pipeline for voxel-based rendering [S1, unofficial]. It builds on the geometry definition and assembly phase, which determines exposed faces and culls internal ones [14339-geometry-definition-of-assembly.md].

## Sources
[S1] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
