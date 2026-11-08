# Alternating Tetrahedral Decomposition

Also called: Alternating C_even/C_odd Decomposition, Checkerboard Tetrahedral Decomposition, Seeding and Decomposition

## Overview
Alternating Tetrahedral Decomposition is a method for converting a structured hexahedral grid into a tetrahedral mesh by grouping cells into packs of five tetrahedra. It solves the topological inconsistency problem inherent in decomposing cubes into tetrahedra, where using a single decomposition pattern for all cells creates T-junctions and discontinuities at shared faces. It sets itself apart from non-alternating methods by ensuring that adjacent cells share common faces, maintaining the connectivity required for correct ray-primitive traversal in volume rendering.

## How it works
**Inputs:** A structured grid of hexahedral cells, where each cell $C_i$ is defined by its 3D integer coordinates $(C_x, C_y, C_z)$ and its 8 corner vertices.
**Outputs:** A set of tetrahedra, where each hexahedral cell is decomposed into 5 tetrahedra.

**Data Structures:**
*   **Cell Coordinates:** Integer tuple $(C_x, C_y, C_z)$ identifying the position in the structured grid.
*   **Vertex Indices:** Each hexahedral cell has 8 vertices. The standard indexing for a cubic cell is fixed as shown in Figure 3.1 of [S1, unofficial]:
    *   Vertex 0: $(0,0,0)$ relative
    *   Vertex 1: $(1,0,0)$ relative
    *   Vertex 2: $(0,1,0)$ relative
    *   Vertex 3: $(1,1,0)$ relative
    *   Vertex 4: $(0,0,1)$ relative
    *   Vertex 5: $(1,0,1)$ relative
    *   Vertex 6: $(0,1,1)$ relative
    *   Vertex 7: $(1,1,1)$ relative

**Steps:**
1.  **Classify Cell Parity:** For each cell $C_i$ with coordinates $(C_x, C_y, C_z)$, calculate the parity sum:
    $$ P = (C_x + C_y + C_z) \bmod 2 $$
2.  **Select Decomposition Scheme:**
    *   If $P = 0$, use the $C_{\text{even}}$ scheme.
    *   If $P = 1$, use the $C_{\text{odd}}$ scheme.
3.  **Generate Tetrahedra:** Decompose the hexahedral cell into 5 tetrahedra based on the selected scheme. Both schemes consist of 4 corner tetrahedra and 1 center tetrahedron.
    *   **$C_{\text{even}}$ Scheme:**
        *   Corner Tetrahedra vertices: $\{1, 2, 4, 7\}$ (Note: These are the indices of the *corner* tetrahedra, meaning each corner tetrahedron uses a subset of the 8 vertices. Specifically, the corner tetrahedra are formed by connecting the center vertex to the corners. The text specifies "corner tetrahedra at vertex IDs 1, 2, 4, 7". In the context of Garrity's decomposition [S2, unofficial], the 5 tetrahedra are typically formed by a central vertex and the faces. However, [S1, unofficial] explicitly states the corner tetrahedra are at IDs 1, 2, 4, 7 for $C_{\text{even}}$ and 0, 3, 5, 6 for $C_{\text{odd}}$. This implies the "center" tetrahedron is distinct. Let's map this to standard Garrity decomposition where a central point is often used, but here it seems the 5th tetrahedron is the "cell center". In a standard 8-vertex cube, a "center" tetrahedron usually implies a central point. However, [S1, unofficial] says "The fifth tetrahedron is always the cell center." and "splits the cell into four corner tetrahedra and a center one." This suggests a 9-vertex structure or a specific interpretation. Let's look at [S2, unofficial]: "Garrity decomposes each hexahedral cell into five tetrahedra". Garrity's method [Gar90] typically uses the 8 vertices and forms 5 tetrahedra by splitting the cube. A common way is to pick a central vertex (e.g., 0) and form tetrahedra with the faces not containing 0, but that yields 5. Another is using a central point. [S1, unofficial] says "The difference is the orientation of the center tetrahedron and the indices of the corners." This implies the 5th tetrahedron is defined by the geometry of the cell center. Without a explicit central vertex in the input, this likely refers to the geometric center or a specific vertex-based decomposition. Given the constraint "implement from plain text", and [S1, unofficial] specifying "corner tetrahedra at vertex IDs...", we must assume the tetrahedra are formed by subsets of the 8 vertices.
        *   Actually, a standard 5-tetrahedron decomposition of a cube uses the 8 vertices. One common decomposition (Garrity) picks one vertex (say 0) and forms tetrahedra: (0,1,2,4), (0,2,3,4), (0,2,6,4), (0,2,6,7), (0,2,5,7)? No.
        *   Let's stick strictly to [S1, unofficial]: "building the corner tetrahedra at vertex IDs 1, 2, 4 and 7". This phrasing is ambiguous. Does it mean the tetrahedra *are* the vertices? No, a tetrahedron has 4 vertices. It likely means the *orientation* or the *central vertex* of the corner tetrahedra.
        *   Re-reading [S1, unofficial]: "Figure 3.2 illustrates the problem... two different schemes... split the cell into four corner tetrahedra and a center one."
        *   In many contexts, "corner tetrahedra" refers to tetrahedra that touch the corners of the cube.
        *   Let's assume the standard Garrity decomposition which uses 5 tetrahedra from 8 vertices.
        *   If $C_{\text{even}}$ uses corner IDs 1, 2, 4, 7, it likely means the tetrahedra are defined by these vertices as "apices" or specific configurations.
        *   However, without a figure, we must rely on the parity rule.
        *   **Pseudocode Logic:**
            *   Calculate parity.
            *   If even, generate tetrahedra set $T_{even}$.
            *   If odd, generate tetrahedra set $T_{odd}$.
            *   The exact vertex indices for the 5 tetrahedra for each scheme are not fully enumerated in the text, only the "corner tetrahedra at vertex IDs".
            *   *Self-Correction/Gap Fill [G]:* Since the exact vertex tuples for the 5 tetrahedra are not explicitly listed as `(v0, v1, v2, v3)` in the text, but the "corner tetrahedra at vertex IDs" are given, and [S2, unofficial] cites Garrity [Gar90], a standard implementation of Garrity's 5-tet decomposition for a cube with vertices 0-7 is:
                *   Scheme A (Root 0): (0,1,2,4), (0,2,3,4), (0,2,6,4), (0,2,6,7), (0,2,5,7) is not 5.
                *   Standard Garrity: Pick a vertex, say 0. Tetrahedra: (0,1,2,4), (0,2,3,4), (0,2,6,4), (0,2,6,7), (0,2,5,7) -> This is complex.
                *   Another common one: (0,1,2,3) is a face.
                *   Let's look at the "Center" tetrahedron. If there is a center tetrahedron, it might involve the geometric center. But [S1, unofficial] says "The fifth tetrahedron is always the cell center." This might mean the tetrahedron *at* the center, i.e., formed by the central vertices of the faces? No.
                *   Given the ambiguity and the instruction "Never guess", we will define the selection logic and note that the specific vertex tuples for the 5 tetrahedra follow the $C_{\text{even}}$ or $C_{\text{odd}}$ pattern described by the corner IDs.
                *   *Refined Gap Fill [G]:* In the absence of explicit vertex tuples, the implementer must use the standard Garrity decomposition [Gar90] referenced in [S2, unofficial], selecting the variant that matches the "corner tetrahedra at vertex IDs" description. For $C_{\text{even}}$, the corner tetrahedra are associated with vertices 1, 2, 4, 7. For $C_{\text{odd}}$, with 0, 3, 5, 6. The "center" tetrahedron connects the remaining geometry.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompose_cell(cell_coords, cell_vertices):
    """
    cell_coords: tuple (Cx, Cy, Cz)
    cell_vertices: list of 8 vertices [v0, v1, ..., v7]
    Returns: list of 5 tetrahedra, each tetrahedron is a tuple of 4 vertex indices
    """
    Cx, Cy, Cz = cell_coords
    
    # Calculate parity
    parity = (Cx + Cy + Cz) % 2
    
    if parity == 0:
        # C_even scheme
        # Corner tetrahedra are at vertex IDs 1, 2, 4, 7
        # The fifth tetrahedron is the cell center
        # Note: Exact vertex tuples for the 5 tetrahedra are not explicitly 
        # enumerated in the source text beyond the corner ID association.
        # Implementer should use the standard Garrity decomposition variant 
        # corresponding to C_even.
        tetrahedra = get_garrity_even_tetrahedra(cell_vertices)
    else:
        # C_odd scheme
        # Corner tetrahedra are at vertex IDs 0, 3, 5, 6
        tetrahedra = get_garrity_odd_tetrahedra(cell_vertices)
        
    return tetrahedra

def get_garrity_even_tetrahedra(vertices):
    # Placeholder for the specific 5-tetrahedron decomposition 
    # matching the C_even corner IDs (1, 2, 4, 7).
    # Standard Garrity decomposition for a cube usually involves 
    # selecting a root vertex. The "corner IDs" likely refer to 
    # the vertices that are *not* the root or are part of the 
    # central structure.
    # Without explicit tuples, this function must be implemented 
    # based on the specific Garrity [Gar90] algorithm referenced.
    pass

def get_garrity_odd_tetrahedra(vertices):
    # Placeholder for the specific 5-tetrahedron decomposition 
    # matching the C_odd corner IDs (0, 3, 5, 6).
    pass
```

**Edge Cases and Pitfalls:**
*   **T-Junctions:** Using the same decomposition pattern for all cells leads to T-junctions and discontinuities between adjacent cells, destroying the topology of the tetrahedral mesh [S1, unofficial].
*   **Shared Faces:** Adjacent cells share two common faces (one cell quad) and the center tetrahedron shares faces with its four corner tetrahedra [S1, unofficial]. The alternating scheme ensures these shared faces are consistent.

## Subtopics

### Five-Tetrahedron Decomposition
Decomposing a hexahedral cell into five tetrahedra to enable ray-plane intersection traversal for renderers supporting only tetrahedral meshes. A convex hexahedron can be approximated by five tetrahedra if the renderer supports tetrahedral meshes only [S2, unofficial]. This is the foundational decomposition used by Alternating Tetrahedral Decomposition.

### Twelve-Triangle Decomposition
Decomposing a hexahedral cell into twelve triangles and using barycentric coordinates for data interpolation. This method was proposed by Williams et al. [WCA* 90] as an alternative to tetrahedral decomposition, where data values are interpolated using barycentric coordinates on the triangles [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of cells, as each cell is decomposed independently into a constant number (5) of tetrahedra.
*   **Memory Complexity:** Linear, storing 5 tetrahedra per cell.
*   **Strengths:** Maintains topological consistency across the entire volume, preventing rendering artifacts like T-junctions. Compatible with hardware-accelerated triangle rendering if the tetrahedra are rasterized.
*   **Limitations:** Requires sorting of tetrahedra before rendering (as per the parent method [Projected Tetrahedra](877-projected-tetrahedra.md)). The decomposition is view-independent, but the traversal is view-dependent.
*   **When to use:** When rendering unstructured or structured hexahedral volumes using a renderer that only supports tetrahedral meshes or requires consistent topological connectivity.

## Background
The method is based on the work of Garrity [Gar90] and Williams et al. [WCA* 90]. Garrity decomposed each hexahedral cell into five tetrahedra and performed ray-plane intersections for incremental traversal [S2, unofficial]. Williams decomposed each hexahedral cell into twelve triangles and used barycentric coordinates for interpolation [S2, unofficial]. The alternating scheme ($C_{\text{even}}$ and $C_{\text{odd}}$) was introduced to solve the ambiguity of cube decomposition and prevent T-junctions, as detailed in Sowinski's Master's thesis [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S2] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24 (unofficial)
