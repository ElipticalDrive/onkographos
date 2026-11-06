# TetSplat Pre-processing: Mesh Partitioning

Also called: Partitioning the Input Mesh into the Shell and the Solid

## Overview
This method partitions a large unstructured tetrahedral mesh into two distinct topological components: the **Shell** (the non-convex boundary) and the **Solid** (the internal volume). This separation is required because the Shell is rendered as a standard opaque surface mesh using vertex normals, whereas the Solid is rendered using normals derived from a Constructive Solid Geometry (CSG) probe, as interior tetrahedra are only visible at the intersection surface [S1, unofficial]. It distinguishes itself from sibling methods by explicitly separating boundary rendering from interior volume rendering logic to resolve shading discontinuities at the CSG probe interface.

## How it works
**Inputs:** An unstructured tetrahedral mesh consisting of tetrahedra $Tet_k$.
**Outputs:** Two sets of triangles: the Shell (boundary faces) and the Solid (internal faces, further classified as boundary or internal triangles).

**Data Structures:**
*   `Tri`: An array containing all triangles derived from the tetrahedral mesh. Each element stores a sorted list of vertex indices.

**Algorithm Steps:**

1.  **Triangle Extraction and Sorting:**
    For each tetrahedron $Tet_k$ in the input mesh:
    *   Extract the four constituent triangles, denoted $Tri_{k,i}$ for $i \in \{0, 1, 2, 3\}$.
    *   Sort the vertex indices within the vertex list of each $Tri_{k,i}$ to ensure canonical representation (e.g., ascending order).
    *   Store these sorted triangles in the array `Tri`.

2.  **Global Sorting:**
    Sort the entire `Tri` array based on the indexed vertex list of each triangle. This groups identical triangles (those sharing the same three vertices) together in adjacent positions.

3.  **Shell/Solid Classification:**
    Iterate through each triangle $Tri_k$ in the sorted `Tri` array:
    *   Compare the vertex list of $Tri_k$ with the vertex list of the next triangle $Tri_{k+1}$.
    *   **If the vertex lists are identical:** The triangle is shared by two tetrahedra. Classify $Tri_k$ as part of the **Solid**.
    *   **If the vertex lists are different:** The triangle is a boundary face. Classify $Tri_k$ as part of the **Shell**.
    *   Mark the tetrahedron $Tet_k$ associated with a Shell triangle as a **boundary tetrahedron**.

4.  **Solid Sub-classification:**
    Iterate through each triangle $Tri_k$ classified as part of the **Solid**:
    *   Check if the associated tetrahedron $Tet_k$ is a **boundary tetrahedron** (marked in Step 3).
    *   **If $Tet_k$ is a boundary tetrahedron:** Classify $Tri_k$ as a **boundary triangle in the Solid**.
    *   **If $Tet_k$ is not a boundary tetrahedron:** Classify $Tri_k$ as an **internal triangle in the Solid**.

5.  **Degeneracy Removal:**
    Remove degenerate tetrahedra from the mesh using the following rules [S1, unofficial]:
    *   **Needles:** Tetrahedra without large dihedral angles are removed by collapsing their shortest edges.
    *   **Spindles:** Tetrahedra with a single large dihedral angle are removed by splitting the edge opposite to the large angle and then collapsing the shortest edges.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def partition_mesh(tetrahedra):
    # Step 1: Extract and canonicalize triangles
    Tri = []
    for k, Tetk in enumerate(tetrahedra):
        # Extract 4 triangles from Tetk
        triangles = extract_triangles(Tetk)
        for i, tri in enumerate(triangles):
            # Sort vertex indices for canonical comparison
            sorted_indices = sort_indices(tri.vertex_indices)
            Tri.append({
                'indices': sorted_indices,
                'tet_index': k,
                'local_index': i
            })

    # Step 2: Sort all triangles by vertex indices
    Tri.sort(key=lambda x: x['indices'])

    shell_triangles = []
    solid_triangles = []
    boundary_tetrahedra = set()

    # Step 3: Classify Shell vs Solid
    n = len(Tri)
    for i in range(n):
        current = Tri[i]
        # Check if current triangle is identical to the next one
        is_shared = False
        if i + 1 < n:
            next_tri = Tri[i + 1]
            if current['indices'] == next_tri['indices']:
                is_shared = True
        
        if is_shared:
            # Shared face -> Solid
            solid_triangles.append(current)
        else:
            # Boundary face -> Shell
            shell_triangles.append(current)
            boundary_tetrahedra.add(current['tet_index'])

    # Step 4: Sub-classify Solid triangles
    solid_boundary_triangles = []
    solid_internal_triangles = []
    
    for tri in solid_triangles:
        tet_idx = tri['tet_index']
        if tet_idx in boundary_tetrahedra:
            solid_boundary_triangles.append(tri)
        else:
            solid_internal_triangles.append(tri)

    # Step 5: Remove degenerate tetrahedra (Needles and Spindles)
    # Note: Implementation details for edge collapsing/splitting are not fully specified in source
    # [G]: Standard mesh simplification techniques would be used here.
    clean_tetrahedra = remove_degeneracies(tetrahedra)

    return shell_triangles, solid_boundary_triangles, solid_internal_triangles, clean_tetrahedra
```

**Parameters:**
*   `Tri`: An array containing all triangles derived from the tetrahedral mesh, used for sorting and classification [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Sorting relies on exact vertex index equality. Floating-point precision issues in vertex positions must be resolved during the initial mesh construction or by using a tolerance-based canonicalization before index sorting.
*   **Degeneracy:** The algorithm explicitly handles "Needles" (no large dihedral angles) and "Spindles" (one large dihedral angle). Failure to remove these can lead to rendering artifacts or incorrect normal derivation [S1, unofficial].
*   **Boundary Definition:** The distinction between "boundary triangle in the Solid" and "Shell" is topological. Shell triangles are on the mesh boundary; Solid boundary triangles are internal faces of tetrahedra that touch the mesh boundary.

## Operating characteristics
**Time Complexity:** Dominated by the sorting step. Sorting $N$ triangles takes $O(N \log N)$ time, where $N$ is the total number of triangle faces ($4 \times$ number of tetrahedra). The iteration steps are linear $O(N)$.
**Memory Complexity:** Requires storage for the `Tri` array, which is proportional to the number of tetrahedra.
**Strengths:** Provides a clean topological separation that allows different shading strategies (vertex normals vs. CSG probe normals) for boundary and interior, ensuring visual consistency at the clipping boundary.
**Limitations:** Requires a full pass over the mesh data and sorting. Degeneracy removal adds preprocessing overhead.

## Background
This method originates from the TetSplat paper by Museth and Lombeyda (2004), which introduced a point-based technique for real-time rendering of large tetrahedral meshes. The partitioning is a prerequisite for the hierarchical bounding sphere structure and the CSG probe clipping mechanism described in the parent article [S1, unofficial].

## Sources
[S1] TetSplat: Real-time Rendering and Volume Clipping of Large ... - Museth, https://ken.museth.org/Publications_files/Museth-Lombeyda_VIS04.pdf, fetched 2026-10-28 (unofficial)
