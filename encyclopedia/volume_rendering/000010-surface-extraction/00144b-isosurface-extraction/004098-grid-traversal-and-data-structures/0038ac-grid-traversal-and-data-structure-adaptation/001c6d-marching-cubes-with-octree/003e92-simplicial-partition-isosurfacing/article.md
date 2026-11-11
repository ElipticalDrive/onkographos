# Simplicial Partition Isosurfacing

Also called: Isosurfaces Over Simplicial Partitions of Multiresolution Grids, Octree Simplicial Isosurfacing

## Overview
Simplicial Partition Isosurfacing extracts a manifold, intersection-free isosurface from a scalar function defined over an arbitrary octree by partitioning space into simplices (tetrahedra) dual to minimal octree cells [S1]. Unlike Dual Contouring, which produces non-manifold geometry in ambiguous sign configurations, and Dual Marching Cubes, which can generate self-intersecting surfaces due to non-convex dual cells, this method guarantees manifold topology and prevents tetrahedron inversion, ensuring the surface cannot self-intersect [S1].

## How it works
**Inputs and Outputs**
*   **Input:** An octree structure representing a scalar field $F: \mathbb{R}^3 \rightarrow \mathbb{R}$, and an isosurface value $c$ [S1].
*   **Output:** A set of triangles forming a 2-manifold isosurface [S1].

**Data Structures**
The method operates on the octree's minimal cells [S1]. A minimal $m$-cell is defined as a cell of dimension $m$ that does not contain any other cells of dimension $m$ as a subset [S1].
*   **Minimal 3-cells (Cubes):** Leaf nodes in the octree [S1].
*   **Minimal 2-cells (Faces):** The smaller of the two shared faces of adjacent leaf cubes [S1].
*   **Minimal 1-cells (Edges):** Edges shared by adjacent minimal faces or cubes [G].
*   **Minimal 0-cells (Vertices):** Vertices of the octree not contained in any other minimal cell of higher dimension [G].

**Dual Vertices**
For each minimal $m$-cell, a dual vertex $\bar{p} = \langle p, F(p) \rangle \in \mathbb{R}^4$ is created, where $p \in \mathbb{R}^3$ is the spatial position [S1]. The position $p$ is constrained to lie within its containing cell:
*   Dual vertices for 3-cells (cubes) are constrained to the interior of the cube [S1].
*   Dual vertices for 2-cells (faces) are constrained to the interior of the face [S1].
*   Dual vertices for 1-cells (edges) are constrained to the interior of the edge [S1].
*   Dual vertices for 0-cells (vertices) are fixed at the position of the 0-cell vertex [S1].

For $m > 0$, an additional constraint prevents vertices from lying on the boundary of their containing cell to avoid degenerate tetrahedra and T-intersections [S1]. The vertex is constrained to a sub-region of size $1-\epsilon$ of the containing cell, where $\epsilon$ is a small positive value [S1].

**Simplicial Partition**
Space is partitioned recursively into simplices starting from a single vertex, forming a hierarchy [S1].
1.  A line segment is formed by adding an edge vertex to a vertex [S1].
2.  A triangle is formed by adding a face vertex to the line segment [S1].
3.  A tetrahedron is formed by adding a cube vertex to the triangle [S1].

This creates a tetrahedral mesh where each tetrahedron is defined by four dual vertices corresponding to a minimal 0-cell, 1-cell, 2-cell, and 3-cell that are topologically connected [S1].

**Isosurface Extraction**
The isosurface is the set of points $p$ where $F(p) = 0$ (assuming $c$ is subtracted from $F$) [S1]. Within each tetrahedron, the function $F$ is approximated by linear interpolation between the dual vertices [G]. The isosurface intersects the tetrahedron if the signs of $F$ at the vertices differ [G]. Standard linear interpolation is used to find the intersection points on the edges of the tetrahedron, and these points are connected to form triangles [G].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simplicial_partition_isosurfacing(octree, isosurface_value):
    """
    Extracts a manifold isosurface from an octree using simplicial partitioning.
    """
    epsilon = 1e-6  # Small value to prevent boundary placement [S1]
    dual_vertices = {}

    # Step 1: Identify minimal cells and create dual vertices
    minimal_cells = get_minimal_cells(octree)
    for cell in minimal_cells:
        dim = cell.dimension
        if dim == 0:
            p = cell.position
        else:
            # Constrain to interior by shrinking cell by epsilon [S1]
            p = shrink_cell_to_interior(cell, epsilon)

        # Evaluate function at p, subtracting isosurface value [S1]
        f_val = octree.evaluate(p) - isosurface_value
        dual_vertices[cell.id] = (p, f_val)

    # Step 2: Build simplicial partition (tetrahedra)
    # Hierarchy: 0-cell -> 1-cell -> 2-cell -> 3-cell [S1]
    tetrahedra = []
    for cube in minimal_cells_of_dimension(3):
        # Find the nested minimal cells in the hierarchy [S1]
        face = cube.minimal_face
        edge = face.minimal_edge
        vertex = edge.minimal_vertex

        v_cube = dual_vertices[cube.id]
        v_face = dual_vertices[face.id]
        v_edge = dual_vertices[edge.id]
        v_vertex = dual_vertices[vertex.id]

        tetrahedra.append([v_cube, v_face, v_edge, v_vertex])

    # Step 3: Extract isosurface from tetrahedra
    triangles = []
    for tet in tetrahedra:
        positions = [v[0] for v in tet]
        values = [v[1] for v in tet]

        # Find intersection points on edges where signs differ [G]
        intersections = []
        edges = [(0, 1), (0, 2), (0, 3), (1, 2), (1, 3), (2, 3)]
        for i, j in edges:
            if values[i] * values[j] < 0:
                t = values[i] / (values[i] - values[j])
                p_intersect = lerp(positions[i], positions[j], t)
                intersections.append(p_intersect)

        # Connect intersections to form triangles based on sign configuration [G]
        # Exact connectivity follows the standard Marching Tetrahedra lookup table [G]
        triangles.extend(marching_tetrahedra(intersections, len(intersections)))

    return triangles

def get_minimal_cells(octree):
    """Returns all minimal cells in the octree [G]."""
    return octree.get_all_minimal_cells()

def minimal_cells_of_dimension(dim):
    """Filters minimal cells by dimension [G]."""
    return [c for c in get_minimal_cells(octree) if c.dimension == dim]

def shrink_cell_to_interior(cell, epsilon):
    """Constrains the vertex to a cell of size 1-ε of the containing cell [S1]."""
    # Returning the geometric center guarantees a strictly interior point [G]
    return cell.center

def lerp(a, b, t):
    return a + t * (b - a)

def marching_tetrahedra(intersections, count):
    """Connects intersection points to form triangles [G]."""
    if count == 2:
        return [intersections]
    elif count == 3:
        return [intersections]
    elif count == 4:
        return [intersections[:2], intersections[2:]]
    return []
```

**Parameters**
*   $\epsilon$: A small value used to shrink the containment region for dual vertices of non-zero dimensional cells (faces, edges) to prevent boundary placement. Typical value not specified, but implied to be small enough to maintain feature reproduction while avoiding degeneracy [S1].

**Edge Cases and Pitfalls**
*   **Degenerate Tetrahedra:** If dual vertices lie on the boundary of their containing cells, degenerate tetrahedra may form, leading to T-intersections. The $\epsilon$ constraint mitigates this [S1].
*   **Ambiguous Sign Configurations:** Unlike Dual Contouring, this method handles ambiguous sign configurations by using a simplicial partition dual to minimal cells, guaranteeing manifold topology [S1].
*   **Self-Intersection:** Unlike Dual Marching Cubes, this method guarantees tetrahedra cannot invert, preventing self-intersection [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Guarantees manifold topology and intersection-free surfaces; reproduces sharp and thin features; adaptive polygonalization guided by an error metric [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When a manifold, intersection-free isosurface is required from an adaptive octree, especially for sharp features [S1].
*   **When not to use:** Not given in the sources.

## Background
Simplicial Partition Isosurfacing was introduced by Josiah Manson and Scott Schaefer from Texas A&M University [S1]. The method addresses limitations of Dual Contouring (which produces non-manifold, self-intersecting surfaces in ambiguous sign configurations) and Dual Marching Cubes (which can generate self-intersecting surfaces due to non-convex dual cells) [S1]. It extends the concept of partitioning space along function features to improve piecewise linear approximation, focusing on accurately representing the function to produce an accurate isosurface [S1].

## Sources
[S1] Isosurfaces Over Simplicial Partitions of Multiresolution Grids, http://faculty.cs.tamu.edu/schaefer/research/iso_simplicial.pdf, fetched 2026-10-29
