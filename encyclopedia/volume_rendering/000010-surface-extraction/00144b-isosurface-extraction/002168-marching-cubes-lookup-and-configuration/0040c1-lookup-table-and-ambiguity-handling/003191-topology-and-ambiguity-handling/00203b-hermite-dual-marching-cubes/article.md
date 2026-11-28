# Hermite Dual Marching Cubes

Also called: Dual Marching Cubes with Hermite data, Hermite Dual Contouring, Hermite extension to Dual MC, Nielson's Dual Marching Cubes with QEF

## Overview
Hermite Dual Marching Cubes is a surface extraction method that combines the topological guarantees of Dual Marching Cubes with the feature-preserving capability of Dual Contouring [S1][S2, unofficial]. It solves the problem of non-manifold vertices and edges that arise in standard Dual Contouring when processing ambiguous sign configurations on uniform grids, while simultaneously reproducing sharp features that standard Marching Cubes fails to capture [S1][S2, unofficial]. Unlike standard Dual Contouring, which may produce non-manifold topology, this method always produces a topological manifold by leveraging the dual structure of Nielson's Dual Marching Cubes [S1][S2, unofficial].

## How it works
**Inputs:**
*   A uniform 3D grid of scalar field values $f(v)$ at voxel corners $v$ [S2, unofficial].
*   Hermite data associated with grid edges: for each edge intersecting the isosurface, the intersection point $p$ and the normal vector $n$ (gradient direction) at that point [S1][S2, unofficial].
*   An isovalue $I$ [S2, unofficial].

**Outputs:**
*   A manifold quadrilateral mesh representing the isosurface [S2, unofficial].

**Data Structures:**
*   **Grid:** A 3D array of scalar values [S2, unofficial].
*   **Edge Data:** For each grid edge, store the intersection point $p \in \mathbb{R}^3$ and normal $n \in \mathbb{R}^3$ if the edge crosses the isosurface [S1][S2, unofficial].
*   **Cell Lookup:** A modified Marching Cubes lookup table that identifies cycles of sign-changing edges for all 256 configurations [S2, unofficial].
*   **Vertex Map:** A mapping from each identified cycle to a vertex position [S2, unofficial].

**Method Steps:**

1.  **Identify Sign-Change Edges:** For each voxel cell, determine which of its 12 edges intersect the isosurface. An edge intersects if the scalar values at its endpoints straddle the isovalue $I$ [S2, unofficial].
2.  **Determine Contour Topology:** For each cell, use the modified Marching Cubes lookup table to identify the cycles of intersecting edges [S2, unofficial]. Nielson's Dual Marching Cubes associates one vertex with each cycle of edges within a cell [S2, unofficial].
3.  **Construct Quadratic Error Functions (QEF):**
    *   For each cycle identified in Step 2, collect all Hermite data (intersection points $p_i$ and normal vectors $n_i$) from the edges associated with that cycle [S2, unofficial].
    *   Construct a QEF for the vertex associated with this cycle. The QEF minimizes the distance from the vertex position $x$ to the planes defined by the Hermite data [S1][S2, unofficial].
    *   The QEF is defined as:
        $$ E(x) = \sum_{i} (n_i \cdot (x - p_i))^2 $$
        where the sum is over all edges $i$ associated with the cycle [S1][S2, unofficial].
    *   Alternatively, for the edge-based vertex placement in some interpretations, the QEF is constructed using Hermite data on the edges associated with that vertex [S1].
4.  **Minimize QEF:** Solve the linear system derived from minimizing $E(x)$ to find the optimal vertex position $x^*$ [S1][S2, unofficial]. This is typically done by solving $A^T A x = A^T b$, where $A$ is a matrix of normals and $b$ is a vector of dot products $n_i \cdot p_i$ [S1][S2, unofficial].
5.  **Generate Polygons:**
    *   For each grid edge that intersects the isosurface, identify the vertices associated with that edge in the four adjacent cells sharing that edge [S2, unofficial].
    *   Connect these vertices to form a polygon (typically a quadrilateral) [S2, unofficial].
    *   This creates a surface that is dual to the Marching Cubes surface [S2, unofficial].

**Parameters:**
*   Isovalue $I$: Not given in the sources.
*   QEF weighting: Not given in the sources. Standard QEF assumes uniform weighting.

**Edge Cases and Pitfalls:**
*   **Ambiguous Configurations:** Standard Dual Contouring fails on the 8 ambiguous cases of Marching Cubes, leading to non-manifold topology. Hermite Dual Marching Cubes handles these by using the cycle-based vertex placement of Dual Marching Cubes, which guarantees manifold topology [S2, unofficial].
*   **Precision:** Solving the QEF system requires numerical stability. If the normals are nearly parallel or collinear, the system may be ill-conditioned [G].
*   **Boundary Conditions:** Edges on the grid boundary have fewer adjacent cells. The polygon generation step must handle boundary edges appropriately, potentially creating open boundaries or requiring special handling [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hermite_dual_marching_cubes(grid, isovalue, edge_hermite_data):
    """
    grid: 3D array of scalar values
    isovalue: float
    edge_hermite_data: dict mapping edge_id to (point, normal)
    Returns: list of polygons (each polygon is a list of vertex positions)
    """
    vertices = {}
    polygons = []
    grid_shape = grid.shape

    # Iterate over all cells
    for x in range(grid_shape[0] - 1):
        for y in range(grid_shape[1] - 1):
            for z in range(grid_shape[2] - 1):
                cell_edges = get_cell_edges(x, y, z, grid_shape)
                intersecting_edges = [e for e in cell_edges if edge_intersects_isosurface(e, grid, isovalue)]

                # Step 2: Determine contour topology using modified lookup table
                # The sources do not provide the exact lookup table implementation [G].
                cycles = modified_marching_cubes_lookup(intersecting_edges)

                # Step 3 & 4: Construct and minimize QEF for each cycle
                for cycle in cycles:
                    hermite_data = [edge_hermite_data[e] for e in cycle if e in edge_hermite_data]
                    if not hermite_data:
                        continue

                    # Construct QEF matrix A and vector b
                    # E(x) = sum((n_i . (x - p_i))^2) -> A^T A x = A^T b
                    A = [n for p, n in hermite_data]
                    b = [dot(n, p) for p, n in hermite_data]
                    vertex_pos = solve_least_squares(A, b)

                    # Fix: cycle is a list, which is unhashable. Convert to tuple.
                    cycle_id = tuple(sorted(cycle))
                    vertices[cycle_id] = vertex_pos

    # Step 5: Generate polygons
    for edge in all_intersecting_edges(grid, isovalue, grid_shape):
        adjacent_cells = get_adjacent_cells(edge, grid_shape)
        cell_vertices = []
        for cell in adjacent_cells:
            cycle_id = get_cycle_id_for_edge(edge, cell)
            if cycle_id in vertices:
                cell_vertices.append(vertices[cycle_id])

        if len(cell_vertices) == 4:
            polygons.append(cell_vertices)
        elif len(cell_vertices) == 3:
            polygons.append(cell_vertices)

    return polygons

# Helper functions (standard grid operations) [G]
def get_cell_edges(x, y, z, shape):
    verts = [(x,y,z), (x+1,y,z), (x,y+1,z), (x+1,y+1,z), (x,y,z+1), (x+1,y,z+1), (x,y+1,z+1), (x+1,y+1,z+1)]
    return [(verts[0],verts[1]), (verts[0],verts[2]), (verts[0],verts[4]), (verts[1],verts[3]), (verts[1],verts[5]), (verts[2],verts[3]), (verts[2],verts[6]), (verts[3],verts[7]), (verts[4],verts[5]), (verts[4],verts[6]), (verts[5],verts[7]), (verts[6],verts[7])]

def edge_intersects_isosurface(edge, grid, isovalue):
    v1, v2 = edge
    return (grid[v1] - isovalue) * (grid[v2] - isovalue) < 0

def all_intersecting_edges(grid, isovalue, shape):
    edges = {}
    for x in range(shape[0] - 1):
        for y in range(shape[1] - 1):
            for z in range(shape[2] - 1):
                for e in get_cell_edges(x, y, z, shape):
                    if edge_intersects_isosurface(e, grid, isovalue):
                        edges[e] = e
    return edges.keys()

def get_adjacent_cells(edge, shape):
    # Standard grid traversal to find cells sharing an edge [G]
    return []

def get_cycle_id_for_edge(edge, cell):
    # Maps edge to its cycle within a specific cell [G]
    return None

def modified_marching_cubes_lookup(intersecting_edges):
    # Returns list of cycles (list of edges) based on sign configuration [G]
    return [intersecting_edges] if intersecting_edges else []

def solve_least_squares(A, b):
    # Solves min ||Ax - b||^2 using normal equations [G]
    import numpy as np
    A_mat = np.array(A)
    b_vec = np.array(b)
    return np.linalg.lstsq(A_mat, b_vec, rcond=None)[0]

def dot(a, b):
    return sum(x * y for x, y in zip(a, b))
```

## Subtopics

### Dual Marching Cubes
The topological foundation. It produces a manifold surface by associating vertices with cycles of sign-changing edges in each cell, ensuring topological consistency for ambiguous cases [S2, unofficial]. Link: [Topology and Ambiguity Handling](6547-topology-and-ambiguity-handling.md).

### Dual Contouring
A method that uses Hermite data and QEF minimization to place vertices accurately, preserving sharp features. However, it can produce non-manifold topology on uniform grids. Link: [Dual Contouring](5004-dual-contouring.md) (hypothetical sibling).

### Marching Cubes with Disjoint Patch Decomposition
A sibling method that resolves ambiguities by decomposing the surface into disjoint patches, avoiding the need for dual structures. Link: [Marching Cubes with Disjoint Patch Decomposition](5004-marching-cubes-with-disjoint-patch-decomposition.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Always produces a manifold surface; reproduces sharp features using Hermite data; handles ambiguous configurations robustly [S1][S2, unofficial].
*   **Limitations:** Requires Hermite data (normals and intersection points) on edges; more complex than standard Marching Cubes; may produce more vertices than necessary in flat regions (addressed by adaptive contouring variants) [S1].
*   **When to use:** When manifold topology is required and sharp features must be preserved, especially on uniform grids with ambiguous configurations [S1][S2, unofficial].
*   **When not to use:** When Hermite data is unavailable; when performance is critical and sharp features are not important (use standard Marching Cubes); when adaptive resolution is needed (use adaptive variants) [S1].

## Background
Hermite Dual Marching Cubes was introduced as an extension of Nielson's Dual Marching Cubes algorithm [S1][S2, unofficial]. It was developed to address the limitations of standard Dual Contouring, which produces non-manifold topology on uniform grids for ambiguous sign configurations [S2, unofficial]. The method combines the topological guarantees of Dual Marching Cubes with the feature-preserving capabilities of Dual Contouring [S1][S2, unofficial].

## Sources
[S1] Manifold Dual Contouring, https://doi.org/10.1109/tvcg.2007.1012, fetched 2026-10-26
[S2] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
