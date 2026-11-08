# Marching Cubes Revisited

Also called: Modified Marching Cubes

## Overview
Marching Cubes Revisited is a topologically correct isosurface extraction method for implicit surfaces on structured grids. Unlike the original Lorensen and Cline algorithm, it guarantees correct topology by resolving face ambiguities via specific diagonal connections and enforcing interval constraints that eliminate impossible sign configurations. It serves as a robust foundation for subsequent octree-based meshing.

## How it works
**Inputs:**
- A smooth scalar field function $F: \mathbb{R}^3 \to \mathbb{R}$ defining the implicit surface $S = F^{-1}(0)$ [S1, unofficial].
- A uniform cubic grid.

**Outputs:**
- A triangulated mesh approximating $S$.

**Data Structures:**
- Grid vertices: Uniformly spaced points in $\mathbb{R}^3$.
- Cell configuration: An 8-bit integer representing the sign of $F$ at the 8 vertices of a cell.

**Constraints and Assumptions:**
1.  **Regular Value:** $0$ is a regular value of $F$, meaning $\nabla F \neq 0$ at every point on $S$ [S1, unofficial].
2.  **Vertex Exclusion:** $S$ does not contain any grid vertices ($F(v) \neq 0$ for all vertices $v$) [S1, unofficial].
3.  **Single Intersection:** $S$ intersects each grid edge at most once [S1, unofficial].
4.  **Interval Condition:** For each cell $C$, one of the following must hold:
    $$0 \notin \Box F(C) \quad \vee \quad 0 \notin \langle \Box \nabla F(C), \Box \nabla F(C) \rangle$$
    where $\Box F(C)$ is the interval enclosure of $F$ over $C$, and $\Box \nabla F(C)$ is the interval enclosure of the gradient. The second clause implies that for all $x, y \in C$:
    $$\langle \nabla F(x), \nabla F(y) \rangle > 0$$
    This ensures the surface is parametrizable in at least one axis direction for cells containing the surface [S1, unofficial].

**Algorithm Steps:**
1.  **Evaluate:** Compute $F(v)$ for all grid vertices $v$.
2.  **Classify:** For each cell, determine the sign configuration (14 possible types up to rotation, mirroring, and sign change) [S1, unofficial].
3.  **Filter:** Discard cells where the sign configuration is 5, 8, 12, 13, or 14. These are impossible under the interval condition [S1, unofficial].
    -   *Reasoning:* Configuration 5, for example, implies sign changes in opposite directions on edges $cd$ and $ef$, and non-parametrizability in other axes, contradicting the inner product condition [S1, unofficial].
4.  **Triangulate:** For the remaining 9 configurations:
    -   Place the isosurface vertex at the **center** of the intersected edge, not via linear interpolation. This ensures topological correctness [S1, unofficial].
    -   Resolve ambiguities in configurations 4 and 6. These cases have a single face where the function changes sign along all four edges, creating four vertices on that face. Connect these vertices pairwise such that the resulting segments are parallel to $(1,1,0)$, $(1,0,1)$, or $(0,1,1)$, depending on the face orientation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_revisited(grid_vertices, F, grid_spacing):
    """
    Extracts isosurface from grid using Marching Cubes Revisited.
    Assumes F is smooth, 0 is a regular value, and interval conditions hold.
    """
    mesh = Mesh()
    
    # Precompute signs for all vertices
    signs = [1 if F(v) > 0 else -1 for v in grid_vertices]
    
    for cell in grid.cells:
        # Get signs of the 8 vertices
        cell_signs = [signs[v] for v in cell.vertices]
        
        # Determine configuration index (0-15)
        config = 0
        for i, s in enumerate(cell_signs):
            if s > 0:
                config |= (1 << i)
        
        # Skip impossible configurations (5, 8, 12, 13, 14)
        if config in [5, 8, 12, 13, 14]:
            continue
            
        # Skip empty or full cells (0 and 15)
        if config == 0 or config == 15:
            continue
            
        # Handle ambiguous cases (4 and 6)
        if config == 4 or config == 6:
            # Identify the ambiguous face
            # The face has 4 vertices with mixed signs, others are uniform
            # Connect vertices on this face with diagonals parallel to (1,1,0), (1,0,1), or (0,1,1)
            # depending on face normal orientation
            triangles = resolve_ambiguity(cell, config)
        else:
            # Standard triangulation for non-ambiguous cases
            triangles = standard_triangulation(cell, config)
            
        mesh.add_triangles(triangles)
        
    return mesh

def resolve_ambiguity(cell, config):
    """
    Resolves ambiguity for config 4 or 6 by connecting vertices on the 
    ambiguous face with specific diagonal directions.
    """
    # Identify the face where all 4 edges have sign changes
    # This face has 2 positive and 2 negative vertices
    face_vertices = get_ambiguous_face_vertices(cell)
    
    # Determine face normal/orientation to choose diagonal
    # Diagonals are parallel to (1,1,0), (1,0,1), or (0,1,1)
    # Connect the two positive vertices and two negative vertices? 
    # No, connect pairs such that segments are parallel to the specified vectors.
    # Typically, this means connecting v0-v1 and v2-v3 if they form the correct diagonal.
    
    # Place vertices at edge centers
    v0 = cell.vertices[face_vertices[0]]
    v1 = cell.vertices[face_vertices[1]]
    v2 = cell.vertices[face_vertices[2]]
    v3 = cell.vertices[face_vertices[3]]
    
    # Calculate edge centers for the 4 edges of the face
    # Note: The algorithm places vertices at edge centers of the CELL edges, 
    # not just the face edges. But for the ambiguous face, the vertices 
    # lie on the 4 edges of that face.
    
    # Let's assume standard edge indexing for a cube:
    # Edges: 0:(0,1), 1:(1,3), 2:(3,2), 3:(2,0), 4:(4,5), 5:(5,7), 6:(7,6), 7:(6,4)
    # Face 0 (z=0): vertices 0,1,3,2. Edges 0,1,2,3.
    
    # Get the 4 edge centers on the ambiguous face
    edge_centers = []
    for edge in ambiguous_face_edges:
        v_start = cell.vertices[edge[0]]
        v_end = cell.vertices[edge[1]]
        center = (v_start + v_end) / 2.0
        edge_centers.append(center)
        
    # Connect based on orientation
    # If face normal is +Z, connect (0,1) and (2,3) -> parallel to (1,0,0)? No.
    # The rule is: segments parallel to (1,1,0), (1,0,1), (0,1,1).
    # For a face in XY plane (normal Z), the diagonal is (1,1,0) or (1,-1,0).
    # We choose the one that maintains consistency.
    
    # Simplified: Connect v0-v2 and v1-v3 if they form the (1,1,0) type diagonal?
    # Actually, the source says "connect them such that resulting segments are parallel to..."
    # This implies a specific pairing.
    
    # For config 4/6, the 4 vertices are on the face.
    # Let the vertices be A, B, C, D in order around the face.
    # Connect A-C and B-D? Or A-B and C-D?
    # The source specifies parallel to (1,1,0) etc.
    # This corresponds to the main diagonal of the face.
    
    # Implementation detail:
    # If face is XY, connect (0,3) and (1,2) -> vector (1,1,0) or similar.
    # We need to map the specific vertices to the diagonal.
    
    # For brevity, we assume a helper exists that maps the 4 vertices 
    # to the two diagonals and selects the correct one based on face normal.
    return create_triangles_from_diagonals(edge_centers, cell.normal)

def standard_triangulation(cell, config):
    """
    Standard MC triangulation for non-ambiguous configs.
    Vertices placed at edge centers.
    """
    triangles = []
    # Map config to edge indices
    edge_indices = config_to_edges[config]
    
    # Create vertices at edge centers
    vertices = []
    for edge_idx in edge_indices:
        v1 = cell.vertices[edge_idx[0]]
        v2 = cell.vertices[edge_idx[1]]
        center = (v1 + v2) / 2.0
        vertices.append(center)
        
    # Map config to triangle indices (using the vertices list)
    tri_indices = config_to_triangles[config]
    for tri in tri_indices:
        triangles.append((vertices[tri[0]], vertices[tri[1]], vertices[tri[2]]))
        
    return triangles
```

## Operating characteristics
-   **Correctness:** Guarantees topological correctness for surfaces satisfying the interval condition, unlike original MC [S1, unofficial].
-   **Complexity:** Linear in the number of grid cells. Each cell is processed independently.
-   **Limitations:** Requires the interval condition to hold. If the surface is too complex or the grid too coarse, the condition may fail, leading to discarded cells or incorrect topology if not handled. The method assumes single intersections per edge [S1, unofficial].
-   **Performance:** Placing vertices at edge centers is computationally cheaper than linear interpolation, though it may reduce geometric accuracy slightly.

## Background
The method was introduced in "Isotopic approximation of implicit curves and surfaces" [S1, unofficial]. It builds on the original Marching Cubes by Lorensen and Cline [LC87] but modifies the vertex placement and ambiguity resolution to ensure isotopic equivalence (topological correctness). It is often used as a precursor to octree-based meshing algorithms.

## Sources
[S1] Isotopic approximation of implicit curves and surfaces, https://doi.org/10.1145/1057432.1057465, fetched 2026-10-29 (unofficial)
