# Slow Growing Subdivision Connectivity

Also called: SGS Connectivity, Tiered Subdivision

## Overview
Slow Growing Subdivision (SGS) Connectivity is a refinement scheme for volumetric meshes that generalizes Catmull-Clark subdivision rules into a multi-tier factorization [S1]. It solves the problem of defining consistent mesh connectivity for hierarchical volumetric data by decomposing the refinement process into distinct topological steps [S1]. Unlike sibling methods such as Longest Edge Bisection (LEB), which rely on geometric edge lengths to determine refinement, SGS Connectivity is purely topological and deterministic, ensuring that the resulting mesh structure corresponds to a generalized Catmull-Clark subdivision of the base tier 0 cells [S1].

## How it works
**Inputs:** A base volumetric mesh consisting of tier 0 cells (typically hexahedra or tetrahedra), where each face and edge may optionally be tagged as a "feature" [S1].
**Outputs:** A refined mesh with increased vertex count and modified connectivity, indexed by level $l$ and tier $t$, where $(l, 3) \equiv (l+1, 0)$ [S1].

**Data Structures:**
*   **Mesh:** A collection of vertices, edges, and faces.
*   **Tier Index:** Each element is associated with a tier $t \in \{0, 1, 2, 3\}$. Tier 0 elements are the base mesh. Tier 1, 2, and 3 elements are generated during refinement.
*   **Feature Tags:** Boolean flags on faces and edges. If a face is tagged, it remains in the mesh and constrains subdivision to 2D surface rules. If an edge is tagged, it constrains subdivision to 1D [S1].

**Refinement Steps:**
The refinement from level $l$ to $l+1$ is performed in three tiers. Let $V_0$ be the set of vertices in the tier 0 mesh.

1.  **Tier 0 to Tier 1 (Cell Center Insertion):**
    *   Insert a new vertex $v_{new}$ at the centroid of each tier 0 cell [S1].
    *   Connect $v_{new}$ to all vertices of the tier 0 cell, forming a set of pyramids [S1].
    *   Merge pairs of pyramids that share a face belonging to the tier 0 mesh [S1].
    *   Remove the shared tier 0 faces from the mesh connectivity [S1].
    *   *Result:* The mesh now contains tier 1 cells. If a tier 0 face was tagged as a feature, it is preserved, and subdivision on that face uses only the face's vertices (2D surface subdivision) [S1].

2.  **Tier 1 to Tier 2 (Edge Merging):**
    *   Identify all newly created pyramids that share an edge belonging to the tier 0 mesh [S1].
    *   Merge these pyramids along the shared tier 0 edge to form tier 2 cells [S1].
    *   *Result:* The connectivity begins to resemble the generalized Catmull-Clark structure around tier 0 edges [S1].

3.  **Tier 2 to Tier 3 (Vertex Merging and Edge Deletion):**
    *   Insert vertices into tier 2 cells (which correspond to tier 0 edges) [S1].
    *   Construct pyramids by connecting these new vertices to the vertices of the tier 2 cells [S1].
    *   Merge all cells that share a vertex from the original tier 0 mesh $V_0$ [S1].
    *   Delete all edges present in the tier 1 and tier 2 cells [S1].
    *   *Result:* The final connectivity corresponds to the generalized Catmull-Clark subdivision applied to the tier 0 cells [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slow_growing_subdivision(mesh):
    """
    Performs one level of Slow Growing Subdivision connectivity refinement.
    mesh: Object containing vertices, faces, edges, and feature tags.
    Returns: Refined mesh with new vertices and updated connectivity.
    """
    new_vertices = []
    new_faces = []
    new_edges = []
    
    # Tier 0 -> Tier 1: Cell refinement
    for cell in mesh.tier_0_cells:
        # Insert centroid vertex
        centroid = compute_centroid(cell.vertices)
        new_vertices.append(centroid)
        
        # Form pyramids: connect centroid to each face of the cell
        for face in cell.faces:
            # Check if face is a feature
            if face.is_feature:
                # Feature faces remain; subdivision is 2D
                # Logic for 2D surface subdivision would be applied here
                # For connectivity, we keep the face and add edges from centroid to face vertices
                for v in face.vertices:
                    new_edges.append((centroid, v))
            else:
                # Non-feature faces are removed/merged
                # We create pyramid faces connecting centroid to the face edges
                # This effectively merges pyramids sharing the tier 0 face
                pass # Merging logic is implicit in the removal of tier 0 faces
        
        # Remove shared tier 0 faces from the active mesh
        for face in cell.faces:
            if not face.is_feature:
                mesh.remove_face(face)

    # Tier 1 -> Tier 2: Edge merging
    # Identify pyramids sharing tier 0 edges
    tier_0_edges = mesh.tier_0_edges
    for edge in tier_0_edges:
        # Find all pyramids (cells) sharing this edge
        sharing_cells = find_cells_sharing_edge(edge, mesh.current_cells)
        
        # Merge these cells into tier 2 cells
        # The specific topology of the merge depends on the base cell type
        # For hexahedra, this creates a structure around the edge
        merge_cells_into_tier_2(sharing_cells, edge)

    # Tier 2 -> Tier 3: Vertex merging and cleanup
    # Insert vertices into tier 2 cells (associated with tier 0 edges)
    for tier_2_cell in mesh.tier_2_cells:
        centroid = compute_centroid(tier_2_cell.vertices)
        new_vertices.append(centroid)
        
        # Construct pyramids
        for v in tier_2_cell.vertices:
            new_edges.append((centroid, v))

    # Merge all cells sharing a tier 0 vertex
    for v0 in mesh.tier_0_vertices:
        sharing_cells = find_cells_sharing_vertex(v0, mesh.current_cells)
        merge_cells_around_vertex(sharing_cells, v0)

    # Delete edges present in tier 1 and tier 2 cells
    for edge in mesh.tier_1_edges:
        mesh.remove_edge(edge)
    for edge in mesh.tier_2_edges:
        mesh.remove_edge(edge)

    return Mesh(new_vertices, new_faces, new_edges)
```

**Parameters:**
*   **Feature Tags:** Boolean flags on faces and edges. Feature faces remain in the mesh and constrain subdivision to 2D surface subdivision. Feature edges remain in the mesh and constrain subdivision to 1D [S1].

**Edge Cases and Pitfalls:**
*   **Feature Tags:** If a face is tagged as a feature, the standard 3D merging rules are overridden, and 2D surface subdivision rules are applied to that face [S1].
*   **Degenerate Input:** The method assumes valid tier 0 cells. Degenerate cells (zero volume) may cause undefined behavior during centroid calculation or pyramid formation [G].
*   **Precision:** While connectivity is topological, subsequent position updates (not covered here) may suffer from precision issues if not handled carefully [G].

## Subtopics

### [Generalized Linear B-Spline Wavelets](7672-generalized-linear-b-spline-wavelets.md)
Generalized Linear B-Spline Wavelets is a wavelet transform designed for progressive compression of volumetric subdivision meshes, specifically those generated by Slow Growing Subdivision (SGS) connectivity. It solves the problem of representing hierarchical volumetric data with controllable error tolerances by computing wavelet coefficients that capture the difference between predicted cell centers (centroids) and actual inserted vertices. Unlike the alternative SGS Wavelet Transform, which smooths original vertices using neighbor positions, Generalized Linear B-Spline Wavelets smooths original vertices exclusively through the influence of wavelet coefficients, resulting in superior compression efficiency at low bit rates.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Scaling:** The scheme generalizes to arbitrary dimensions, though the paper focuses on the 3D case [S1].
*   **Strengths:** Provides a deterministic connectivity structure that generalizes Catmull-Clark subdivision, enabling the use of wavelet transforms for compression [S1].
*   **Limitations:** The complexity of the multi-tier merging operations may be higher than simpler bisection methods [G].
*   **When to use:** When using wavelet-based compression on volumetric subdivision meshes [S1].
*   **When not to use:** When geometric adaptivity based on edge length is required (use LEB instead) [S1].

## Background
Slow Growing Subdivision was introduced in the context of 3D data processing and visualization [S1]. It generalizes B-spline wavelets to n-D subdivision for regular grids, building on lifted wavelets by Bertram [S1]. The scheme was presented at the 2nd International Symposium on 3D Data Processing, Visualization, and Transmission (3DPVT’04) [S1]. It is used for progressive compression of scientific data sets defined on regular grids [S1].

## Sources
[S1] Progressive Compression of Volumetric Subdivision Meshes, https://digital.library.unt.edu/ark:/67531/metadc1405535/?q=volume data compression rendering, fetched 2026-10-28
