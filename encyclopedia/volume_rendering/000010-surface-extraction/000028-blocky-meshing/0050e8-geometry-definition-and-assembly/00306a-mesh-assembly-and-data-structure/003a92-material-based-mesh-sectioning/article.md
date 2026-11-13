# Material-Based Mesh Sectioning

Also called: GenerateTriangles, Mesh Splitting by Material

## Overview
Material-Based Mesh Sectioning splits a decoded mesh into separate arrays of triangles based on the material ID associated with each triangle's vertices [S1, unofficial]. It solves the problem of organizing geometry for multi-material rendering by ensuring that triangles sharing the same material are grouped contiguously, which facilitates subsequent merging and GPU resource management [S1, unofficial]. It differs from sibling methods like [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md) by focusing on the logical disambiguation and caching of material references rather than the geometric assembly or sectioning of the mesh itself [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A decoded mesh object containing vertex positions, indices, and per-vertex material IDs.
*   **Output:** An array of mesh sections, where each section corresponds to a unique material ID and contains an array of triangles (triplets of vertices).

**Data Structures**
*   **Mesh Section:** A structure containing an array of triangles. The array is dynamically resized based on the maximum material ID encountered.
*   **Triangle:** Defined by three vertices. Each vertex includes a position (3 floats) and inherits the material ID from the source mesh's vertex data.

**Steps**
1.  **Decode:** Retrieve the decoded mesh using `GetDecodedMesh` [S1, unofficial].
2.  **Sanity Check:** If the number of indices in the mesh is less than 3, return an empty mesh sections array [S1, unofficial].
3.  **Initialize Sections:** Create an empty array to hold mesh sections. The size of this array will be determined by the maximum material ID encountered.
4.  **Iterate Triangles:** Loop through the mesh indices in steps of 3. For each triplet of indices $(i, i+1, i+2)$:
    a.  **Retrieve Vertices:** Get the three vertices corresponding to the current indices.
    b.  **Adjust Position:** Add the mesh's global offset to each vertex's position to ensure correct world-space coordinates [S1, unofficial].
    c.  **Determine Material ID:**
        i.  Start with the material ID of Vertex 0.
        ii. If Vertex 1 and Vertex 2 have the same material ID, use that ID instead. This rule prioritizes consistency when two vertices share a material, implying a preference for the majority material in a triangle [S1, unofficial].
    d.  **Ensure Section Existence:** If the determined `materialID` is greater than or equal to the current number of sections, resize the sections array to `materialID + 1`, initializing new elements as empty [S1, unofficial].
    e.  **Add Triangle:** Append the current triangle (with its adjusted positions and determined material ID) to the section at index `materialID` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def material_based_mesh_sectioning(decoded_mesh):
    # Step 1: Decode
    mesh = decoded_mesh
    
    # Step 2: Sanity Check
    if len(mesh.indices) < 3:
        return []
    
    # Step 3: Initialize Sections
    # We don't know the max material ID yet, so we start with an empty list
    sections = []
    
    # Step 4: Iterate Triangles
    for i in range(0, len(mesh.indices), 3):
        idx0, idx1, idx2 = mesh.indices[i], mesh.indices[i+1], mesh.indices[i+2]
        
        # Retrieve Vertices
        v0 = mesh.vertices[idx0]
        v1 = mesh.vertices[idx1]
        v2 = mesh.vertices[idx2]
        
        # Adjust Position (add mesh offset)
        v0.pos += mesh.offset
        v1.pos += mesh.offset
        v2.pos += mesh.offset
        
        # Determine Material ID
        mat_id = v0.material_id
        if v1.material_id == v2.material_id:
            mat_id = v1.material_id
        
        # Ensure Section Existence
        if mat_id >= len(sections):
            # Resize to mat_id + 1, initializing new elements as empty
            sections.extend([[] for _ in range(mat_id - len(sections) + 1)])
        
        # Add Triangle
        # A triangle is represented as a tuple of vertices
        triangle = (v0, v1, v2)
        sections[mat_id].append(triangle)
        
    return sections
```

**Parameters**
*   Not given in the sources.

**Edge Cases and Pitfalls**
*   **Degenerate Input:** If the mesh has fewer than 3 indices, the function returns an empty array [S1, unofficial].
*   **Material ID Consistency:** The rule for determining material ID (preferring V1/V2 if they match) may lead to unexpected material assignments if a triangle has vertices with IDs (A, B, B). In this case, the triangle is assigned material B, even if V0 is A. This behavior is specific to the source implementation [S1, unofficial].
*   **Sparse Material IDs:** If material IDs are sparse (e.g., IDs 0 and 1000 are used, but 1-999 are not), the sections array will be large and mostly empty. This is a memory inefficiency not addressed by the basic algorithm [S1, unofficial].

## Subtopics

### [Material ID Assignment & Caching](13805-material-id-assignment-caching.md)
Maps block-level material identifiers to unique, globally distinct terrain material IDs to facilitate mesh merging and GPU resource management. It creates material objects on demand and caches the results to avoid redundant GPU allocations, ensuring that visually identical materials share a single GPU resource. It differs from sibling methods by focusing on the logical disambiguation and caching of material references rather than the geometric assembly or sectioning of the mesh itself.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of triangles in the input mesh. Each triangle is processed exactly once.
*   **Memory Complexity:** $O(N + M)$, where $N$ is the number of triangles and $M$ is the maximum material ID. The sections array size depends on $M$, and the total number of triangles stored is $N$.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with the number of triangles. Memory usage scales with the number of triangles and the range of material IDs.
*   **Strengths:** Simple to implement; effectively groups geometry by material for efficient rendering.
*   **Limitations:** Inefficient for sparse material IDs; does not handle material blending or transparency sorting.
*   **When to Use:** When rendering meshes with distinct, non-overlapping materials per triangle.
*   **When Not to Use:** When material IDs are sparse or when material blending is required.

## Background
The method originates from the VoxelGenerator project by Jay2645 [S1, unofficial]. It is used in voxel-based game engines to manage terrain geometry, where different block types (grass, stone, etc.) are assigned different materials. The method ensures that triangles for each material type are grouped together, allowing for efficient batch rendering.

## Sources
[S1] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
