# View-Dependent Subdivision

Also called: Adaptive Tetrahedralization, Dynamic Mesh Refinement, Selective Tetrahedral Subdivision

## Overview
View-Dependent Subdivision is a strategy to limit the computational overhead of full tetrahedral subdivision by only subdividing cells that are intersected by a viewing ray in more than one segment [S1]. It serves as a refinement step within the broader framework of handling non-planar faces in curvilinear grids, ensuring that only "problem cells" undergo the expensive conversion to tetrahedral representation while leaving other cells as quadrilaterals [S1][S2, unofficial]. This approach balances rendering quality and performance by adapting the mesh resolution based on view-dependent criteria such as screen-space error or depth [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A curvilinear grid mesh containing hexahedral or quadrilateral cells, a specific viewpoint $V$, and a subdivision criterion (e.g., screen-space error, depth, or intersection multiplicity).
*   **Output:** A hybrid mesh where "problem cells" are subdivided into tetrahedra, and non-problem cells remain as quadrilaterals or are processed via their original projection.

**Data Structures:**
The mesh is represented as a collection of cells. Each cell stores its vertex coordinates and topology. For hexahedral cells, the structure typically includes 8 vertices. When subdivided, the cell is replaced by a set of tetrahedra. The sources note that a single hexahedron subdivided into six tetrahedra increases the polygon count from 7 to 22 and the vertex count from 41 to 66 (counting repetitions for hardware rendering) [S1].

**Steps:**
1.  **Iterate through the mesh:** The engine iterates through each tetrahedron or cell in the unstructured volume data [S2, unofficial].
2.  **Evaluate Subdivision Criterion:** For each cell $A$ and viewpoint $V$, determine if it is a "problem cell."
    *   **Intersection Test:** Check if a viewing ray intersects the cell in more than one segment [S1].
    *   **Problem Face Detection:** For a cell $A$ with a quadrilateral face $F$ divided into triangles $S$ and $T$ by a diagonal from vertex $L$, $F$ is a problem face if:
        1.  The outward normal to $S$ has a positive dot product with the vector $\vec{VL}$ (where $V$ is the viewpoint and $L$ is the vertex).
        2.  The outward normal to $T$ has a negative dot product with $\vec{VL}$.
        3.  Both $V$ and the interior of triangle $S$ lie on the same side of the plane of triangle $T$ [S1].
    *   **Assumption:** This detection assumes distortion is mild enough that multiple intersections occur only when the viewing ray intersects a single curved face more than once [S1].
3.  **Subdivide if Necessary:** If the criterion is met (e.g., the cell is a problem cell or contributes significantly to the image), subdivide the cell into smaller tetrahedra [S2, unofficial].
    *   For a hexahedron, this typically involves dividing it into six tetrahedra [S1].
    *   New vertices are computed based on the subdivision scheme [S2, unofficial].
4.  **Update Mesh:** The mesh structure is updated to reflect the new tetrahedral elements [S2, unofficial].
5.  **Scan Conversion and Compositing:** The geometry is passed to the scan conversion and compositing stages. The subdivision ensures that only relevant parts of the volume are rendered at high detail [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def view_dependent_subdivision(mesh, viewpoint):
    """
    Subdivides problem cells in the mesh based on the viewpoint.
    """
    for cell in mesh.cells:
        if is_problem_cell(cell, viewpoint):
            subdivided_tets = subdivide_hexahedron(cell)
            mesh.replace_cell(cell, subdivided_tets)
    return mesh

def is_problem_cell(cell, viewpoint):
    """
    Checks if a cell is a problem cell by detecting multiple ray intersections.
    Assumes mild distortion.
    """
    # Check if any viewing ray intersects the cell in more than one segment
    # This is equivalent to checking for problem faces as defined in [S1]
    for face in cell.faces:
        if is_problem_face(face, cell, viewpoint):
            return True
    return False

def is_problem_face(face, cell, viewpoint):
    """
    Determines if a quadrilateral face is a problem face.
    Face is divided into triangles S and T by diagonal from vertex L.
    """
    # Identify vertex L and triangles S, T
    L, S, T = decompose_quadrilateral(face)
    
    # Vector from viewpoint to vertex L
    VL = L - viewpoint
    
    # Outward normals
    n_S = outward_normal(S)
    n_T = outward_normal(T)
    
    # Condition 1: n_S . VL > 0
    cond1 = dot(n_S, VL) > 0
    
    # Condition 2: n_T . VL < 0
    cond2 = dot(n_T, VL) < 0
    
    # Condition 3: V and interior of S are on the same side of plane of T
    # Plane of T defined by normal n_T and point on T (e.g., a vertex)
    # Distance of V from plane T
    dist_V = dot(n_T, viewpoint - T.vertex)
    # Distance of a point inside S from plane T
    # Since S and T share an edge, we can check the third vertex of S
    V_S = S.vertex_not_in_shared_edge
    dist_S = dot(n_T, V_S - T.vertex)
    
    cond3 = (dist_V * dist_S) > 0
    
    return cond1 and cond2 and cond3

def subdivide_hexahedron(hex_cell):
    """
    Subdivides a hexahedron into six tetrahedra.
    """
    # Standard subdivision into 6 tetrahedra
    # Vertices of hexahedron: v0..v7
    # Tetrahedra: (v0,v1,v2,v3), (v0,v2,v3,v4), etc.
    # Exact connectivity depends on the specific subdivision scheme.
    # Not given in the sources. [G]
    return generate_tetrahedra(hex_cell.vertices)
```

**Parameters:**
*   **Subdivision Criterion:** Can be based on screen-space error, depth, or intersection multiplicity [S2, unofficial].
*   **Maximum Depth:** The process is recursive or iterative until the desired resolution is reached or a maximum depth is exceeded [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The dot product checks for problem faces rely on precise normal calculations. Numerical errors can lead to incorrect classification of problem cells [S1].
*   **Boundaries:** The assumption of mild distortion may fail for highly curved or distorted cells, leading to missed problem cells or false positives [S1].
*   **Geometry Overhead:** Full subdivision significantly increases polygon and vertex counts (7 to 22 polygons, 41 to 66 vertices for a hexahedron), increasing rasterization effort [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of problem cells. In the worst case (all cells are problem cells), it approaches the complexity of full tetrahedral subdivision. In the best case (no problem cells), it is minimal.
*   **Memory Complexity:** Increases due to the storage of additional tetrahedral elements for subdivided cells.
*   **Measured Performance:** Full subdivision of a hexahedron into six tetrahedra increases rasterization effort because most pixels in the projection are covered by multiple triangles [S1].
*   **Strengths:** Reduces computational overhead compared to full tetrahedral subdivision by only processing problematic cells [S1].
*   **Limitations:** Requires additional logic to detect problem cells. May still incur significant overhead if many cells are problematic [S1].
*   **When to Use:** When dealing with curvilinear grids with mild distortion where full tetrahedral subdivision is too expensive [S1].
*   **When Not to Use:** When distortion is severe, or when the grid is already well-behaved and does not require adaptive refinement [S1].

## Background
*   **Origin:** The concept is described in the context of approximate volume rendering for curvilinear and unstructured grids using hardware-assisted polyhedron projection [S1]. It is also part of the LLNL High Accuracy Volume Renderer for unstructured data [S2, unofficial].
*   **History:** The method addresses the challenge of rendering non-planar faces in curvilinear grids, which can cause self-intersection or multiple intersections with viewing rays [S1][S2, unofficial].
*   **Usage:** Used in high-accuracy volume rendering systems for unstructured data, particularly in scientific visualization contexts [S2, unofficial].

## Sources
[S1] Approximate volume rendering for curvilinear and unstructured grids by hardware-assisted polyhedron projection, https://doi.org/10.1002/(sici)1098-1098(2000)11:1<53::aid-ima6>3.0.co;2-7, fetched 2026-10-25
[S2] The LLNL High Accuracy Volume Renderer for Unstructured Data: Capabilities, Current Limits, and Potential for ASCI/VIEWS Deployment, https://digital.library.unt.edu/ark:/67531/metadc1418933/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
