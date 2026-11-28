# Mesh simplification via face coloring

Also called: Cell-based coloring, DMC simplification, Uniform structure coloring, Vertex and face coloring for parallel mesh processing

## Overview
Mesh simplification via face coloring is a parallel method for reducing the element count of dual marching cubes (DMC) meshes by identifying and removing quadrilaterals based on vertex valence patterns [S1]. It solves the problem of excessive mesh density in isosurface extraction by leveraging the uniform structure of volume data to assign colors to cells, vertices, and faces, enabling conflict-free parallel simplification [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A dual marching cubes mesh consisting of quadrilaterals derived from [Dual Mesh Building](1423-dual-mesh-building.md) [S1].
*   **Output:** A simplified mesh with reduced element count and a recomputed halfedge data structure [S1].

**Data Structures:**
*   **Cell Index:** 3D integer tuple $(i, j, k)$ representing the position in the volume grid [S1].
*   **Cell Color ($c_v$):** Integer in $[0, 7]$ derived from the cell index [S1].
*   **Edge Color ($c_e$):** Integer in $[0, 23]$ derived from the cell color and local edge index [S1].
*   **Quadrilateral Color ($c_q$):** Integer in $[0, 4]$ after reduction [S1].
*   **Vertex Valence:** The number of quadrilaterals incident to a vertex [S1].

**Formulas:**
1.  **Cell Color:** Derived from the 3D cell index $(i, j, k)$ using bitwise operations [S1]:
    $$c_v = (i \& 1) \ | \ ((j \& 1) \ll 1) \ | \ ((k \& 1) \ll 2)$$
    This results in 8 distinct colors [S1].

2.  **Edge Color:** Derived from the cell color $c_v$ and the local edge index $e$ incident to vertex $v_0$ [S1]:
    $$c_e = 3 \times c_v + e$$
    where $e \in \{0, 1, 2\}$ corresponds to local edges $e_0, e_3, e_8$ [S1]. This results in 24 distinct edge colors [S1].

3.  **Quadrilateral Color:** Each cell edge corresponds to a maximum of one quadrilateral, so the edge color is transferred directly to the quadrilateral [S1]. A reduction kernel then maps colors $\ge 5$ to the range $[0, 4]$ [S1].

**Numbered Steps:**

1.  **Compute Cell Colors:** For every active cell in the volume grid, compute $c_v$ using the cell color formula [S1].
2.  **Transfer Colors to Vertices:** Vertices inherit colors from their adjacent cells because they only have neighbors in adjacent cells [S1].
3.  **Compute Edge Colors:** For each cell, compute the colors of its edges $e_0, e_3, e_8$ using $c_e = 3 \times c_v + e$ [S1].
4.  **Transfer Colors to Quadrilaterals:** Assign the computed edge color $c_e$ to the quadrilateral associated with that edge [S1].
5.  **Reduce Quadrilateral Colors:** A second kernel processes faces to reduce the number of quadrilateral colors to 5 [S1]. It iterates through all quadrilaterals with colors $\ge 5$ and assigns a new color between 0 and 4 by checking which color does not yet appear in its neighborhood [S1].
6.  **Identify Simplification Candidates:** Identify quadrilaterals based on the valence patterns of their incident vertices [S1].
    *   **Pattern 3-X–3-Y:** Vertices with valence 3, X, 3, Y (where X and Y are other valences) [S1].
    *   **Pattern 3-3-3-3:** All four incident vertices have valence 3 [S1].
7.  **Simplify Elements:** Remove quadrilaterals identified in Step 6 [S1]. For the 3-X–3-Y pattern, vertices are repositioned during simplification [S1]. For the 3-3-3-3 pattern, elements are removed directly [S1].
8.  **Recompute Halfedge Data Structure:** After each simplification step, the halfedge data structure must be recomputed to maintain connectivity integrity [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cell_color(i, j, k):
    """Compute cell color from 3D index using bitwise pattern."""
    return (i & 1) | ((j & 1) << 1) | ((k & 1) << 2)

def compute_edge_color(cell_color, local_edge_index):
    """Compute edge color from cell color and local edge index e in {0, 1, 2}."""
    return 3 * cell_color + local_edge_index

def reduce_quadrilateral_colors(quads):
    """
    Reduce quadrilateral colors to 5 colors.
    quads: list of quadrilaterals, each with a 'color' attribute and 'neighbors' list.
    """
    for quad in quads:
        if quad.color >= 5:
            neighbor_colors = {n.color for n in quad.neighbors}
            new_color = 0
            while new_color in neighbor_colors:
                new_color += 1
            quad.color = new_color
    return quads

def simplify_mesh(mesh):
    """
    Simplify the mesh based on valence patterns and face coloring.
    mesh: Dual marching cubes mesh object with cells, vertices, and quads.
    """
    # [G] Compute vertex valences from mesh connectivity
    vertex_valences = {}
    for quad in mesh.quads:
        for v in quad.vertices:
            vertex_valences[v.id] = vertex_valences.get(v.id, 0) + 1

    quads_to_remove = []
    for quad in mesh.quads:
        valences = [vertex_valences[v.id] for v in quad.vertices]
        if all(v == 3 for v in valences):
            quads_to_remove.append(quad)
        elif valences[0] == 3 and valences[2] == 3:
            quads_to_remove.append(quad)

    for quad in quads_to_remove:
        mesh.remove_quad(quad)

    # [G] Recompute halfedge data structure after simplification
    mesh.recompute_halfedge_structure()
    return mesh
```

**Parameters:**
*   Number of cell colors: 8 [S1].
*   Number of edge colors: 24 [S1].
*   Final number of quadrilateral colors: 5 [S1].

**Edge Cases and Pitfalls:**
*   **Precision and Boundaries:** Boundary cells do not generate quadrilaterals if an entry in the hash table bucket is an invalid index [S1].
*   **Valence Patterns:** The 3-X–3-Y pattern simplification repositions vertices, which increases the deviation from the iso-value [S1].
*   **Quality Trade-off:** Simplification improves quadrilateral quality but reduces triangle quality and increases the number of irregular vertices for triangular meshes [S1].
*   **Halfedge Recomputation:** The halfedge data structure must be recomputed after each simplification step, which is computationally expensive due to global memory access on the GPU [S1].

## Subtopics

### Dual Marching Cubes Table Generation
Generates the lookup tables for dual marching cubes [G]. It sets itself apart by focusing on the topological rules for vertex placement rather than mesh simplification [G]. Choose it when defining the core extraction logic [G]. [Link](2069-dual-marching-cubes-table-generation.md)

### Manifold Dual Marching Cubes
Ensures the resulting mesh is manifold [G]. It sets itself apart by handling non-manifold configurations explicitly [G]. Choose it when topological correctness is critical [G]. [Link](2135-manifold-dual-marching-cubes.md)

### Halfedge Data Structure Construction
Builds the connectivity structure for the mesh [G]. It sets itself apart by providing the necessary data structure for operations like simplification [G]. Choose it when you need to traverse or modify the mesh topology [G]. [Link](2310-halfedge-data-structure-construction.md)

## Operating characteristics
*   **Time Complexity:** The coloring steps are parallel and efficient [S1]. The halfedge data structure computation is time-consuming due to many global memory accesses on the GPU [S1].
*   **Memory Complexity:** Requires storage for cell, edge, and quadrilateral colors [G]. The reduction step requires temporary storage for neighbor checks [G].
*   **Measured Performance:** For the human skull surface, 352,337 elements with the 3-X–3-Y pattern were simplified, compared to 167,537 with the previous method [S1]. The 3-3-3-3 pattern simplification reduces the total number of elements by 5% [S1].
*   **Scaling:** The method is designed for parallel execution on GPUs, leveraging the uniform structure of volume data [S1].
*   **Strengths:** Significantly reduces element count compared to previous methods [S1]. Improves quadrilateral quality [S1].
*   **Limitations:** Increases deviation from the iso-value for 3-X–3-Y patterns [S1]. Reduces triangle quality [S1]. Requires expensive halfedge recomputation [S1].
*   **When to Use:** When a high-quality quad-only mesh is desired and element count reduction is important [G].
*   **When Not to Use:** When strict adherence to the iso-value is critical (due to vertex repositioning in 3-X–3-Y patterns) or when triangle quality is more important than quad quality [S1].

## Background
The method was introduced in a parallel dual marching cubes approach to quad-only surface reconstruction [S1]. It builds on the uniform structure of volume data to enable efficient parallel processing [S1]. The method was evaluated on various datasets including torso, body, synthetic iWP, and gen2 data sets [S1].

## Sources
[S1] A parallel dual marching cubes approach to quad only surface reconstruction, https://doi.org/10.1007/s00371-021-02139-w, fetched 2026-10-26
