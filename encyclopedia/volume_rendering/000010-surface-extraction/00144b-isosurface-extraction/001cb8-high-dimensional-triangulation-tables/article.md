# High-Dimensional Triangulation Tables

Also called: Bhaniramka et al. triangulation algorithm

## Overview
High-Dimensional Triangulation Tables is a method for generating precomputed triangulation tables for isosurface extraction in $n$-dimensional hypercubes [S1, book]. It addresses the excessive simplex count produced by recursive contour-meshing, which requires subdividing 4-cells into 4-simplices before contouring, analogous to breaking a cube into five or six tetrahedra in 3D [S1, book]. Unlike recursive decomposition, this method constructs an $n$-polytope via convex hulls to determine the isosurface directly within the hypercube, ensuring consistent triangulation between adjacent faces [S1, book].

## How it works
**Inputs:**
*   Scalar field values $V_i$ at the $2^n$ vertices of an $n$-dimensional hypercube cell [S1, book].
*   A scalar isovalue $I$ [S1, book].

**Outputs:**
*   A set of simplices representing the isosurface intersection within the cell [S1, book].
*   A triangulation table mapping vertex configurations to edge intersections and triangulations [S1, book].

**Data Structures:**
*   **Vertex Configuration Index:** An integer formed by $2^n$ bits [S1, book]. For vertex $i$ (where $i \in [0, 2^n - 1]$), the $i$-th bit is set to 1 if $V_i > I$, and 0 otherwise [S1, book].
*   **Triangulation Table:** A lookup structure indexed by the Vertex Configuration Index [S1, book]. Each entry contains a list of triangles (triplets of vertex indices or edge intersection points) [S1, book].

**Algorithm Steps:**

1.  **Label Vertices:** For each vertex $v_i$ of the hypercube with scalar value $V_i$, assign a positive label if $V_i > I$, and a negative label if $V_i < I$ [S1, book].
2.  **Identify Intersecting Edges:** Collect the midpoints of all edges of the hypercube where one endpoint has a positive label and the other has a negative label [S1, book].
3.  **Construct Convex Hull:** Create the convex hull of the collected midpoints together with other vertices in the hypercube that share the same label (positive or negative), forming an $n$-polytope within the hypercube [S1, book].
4.  **Extract Isosurface Faces:** Remove any faces of the polytope that lie on the boundary of the hypercube (faces sharing vertices of the hypercube) [S1, book]. The remaining faces comprise the isosurface [S1, book].
5.  **Triangulate:** If the remaining faces are not simplices, perform triangulation, ensuring consistent triangulation between adjacent faces of the hypercube [S1, book].
6.  **Table Generation (Offline):** The above steps are performed for all $2^{2^n}$ possible vertex configurations to build the triangulation table, which is created only once for a given $n$ dimension [S1, book].
7.  **Runtime Extraction:** For each cell, encode cell vertex values relative to the iso-value using one bit per vertex (1 if value > iso-value, 0 otherwise) to index the triangulation table, then use the table to decide which edges intersect with the isosurface for the given cell configuration [S1, book].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_vertex_index(vertices, isovalue):
    """
    Computes the integer index for the triangulation table.
    vertices: list of scalar values at the 2^n vertices of the hypercube.
    isovalue: the threshold scalar value.
    Returns: integer index where bit i is 1 if vertices[i] > isovalue.
    """
    index = 0
    for i, v in enumerate(vertices):
        if v > isovalue:
            index |= (1 << i)
    return index

def generate_triangulation_table(n):
    """
    Generates the triangulation table for an n-dimensional hypercube.
    This is an offline process.
    """
    num_vertices = 2 ** n
    table_size = 2 ** num_vertices
    table = [None] * table_size
    
    # Precompute hypercube edge connectivity: list of (v1_idx, v2_idx)
    edges = []
    for i in range(num_vertices):
        for j in range(i + 1, num_vertices):
            if bin(i ^ j).count('1') == 1:
                edges.append((i, j))
                
    for config in range(table_size):
        # Determine vertex labels from config bits
        labels = [(config >> i) & 1 for i in range(num_vertices)]
        
        # Step 2: Collect midpoints of edges with mixed labels
        intersecting_midpoints = []
        for v1, v2 in edges:
            if labels[v1] != labels[v2]:
                intersecting_midpoints.append((v1 + v2) / 2.0)
                
        # Step 3: Create convex hull of midpoints + consistent vertices
        consistent_vertices = [i for i, l in enumerate(labels) if l == 1]
        all_points = intersecting_midpoints + consistent_vertices
        polytope_faces = compute_n_d_convex_hull(all_points)
        
        # Step 4: Remove faces on the hypercube boundary
        isosurface_faces = [f for f in polytope_faces if not is_on_hypercube_boundary(f, num_vertices)]
        
        # Step 5: Triangulate non-simplex faces consistently
        triangles = triangulate_polygon_mesh(isosurface_faces)
        
        table[config] = triangles

    return table

def compute_n_d_convex_hull(points):
    """
    Standard n-dimensional convex hull algorithm (e.g., QuickHull).
    Returns a list of faces, where each face is a list of point indices.
    """
    # Implementation uses standard computational geometry routines
    # to compute the boundary facets of the convex hull of points.
    return standard_convex_hull_routine(points)

def is_on_hypercube_boundary(face, num_vertices):
    """
    Checks if a face lies on the boundary of the hypercube.
    A face is on the boundary if all its vertices lie on a single 
    (n-1)-dimensional facet of the hypercube.
    """
    return check_facet_membership(face)

def triangulate_polygon_mesh(faces):
    """
    Standard triangulation algorithm for non-simplex faces.
    Ensures consistent vertex ordering and shared edge handling.
    """
    return standard_triangulation_routine(faces)

def extract_isosurface_cell(vertices, isovalue, table):
    """
    Extracts the isosurface for a single cell using the precomputed table.
    """
    index = compute_vertex_index(vertices, isovalue)
    return table[index]
```

**Parameters:**
*   $n$: The dimension of the hypercube [S1, book].
*   Table Size: $2^{2^n}$ [S1, book]. For $n=3$, size is 256. For $n=4$, size is 65,536. For $n=5$, size is over 4 billion [S1, book].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons near the isovalue must be consistent to avoid gaps or overlaps [S1, book].
*   **Boundary Faces:** Correctly identifying and removing faces that lie on the hypercube boundary is critical for mesh coherence across cells [S1, book].
*   **Triangulation Consistency:** Ensuring that adjacent cells share the same triangulation for shared faces is essential for a watertight mesh [S1, book].

## Subtopics

### Recursive n-simplex subdivision
A technique that uses a recursive method to split $n$-dimensional cells into $n$-simplexes [S2, unofficial]. The resulting surfaces exist in four dimensions and are displayed using stereographic projection or slicing to reduce dimensionality [S2, unofficial]. The centroid division technique used to break cells into simplexes is suboptimal but can be improved using lookup tables similar to those used in marching cubes [S2, unofficial]. It was used to compute swept volumes of time-varying data generated from electromagnetic field simulations, allowing them to be displayed as animations [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Offline table generation is exponential in $n$ [S1, book]. Runtime extraction is $O(1)$ per cell after table lookup, assuming the table fits in memory [S1, book].
*   **Memory Complexity:** The table size is $2^{2^n}$ [S1, book]. This becomes prohibitive for $n \ge 5$ (over 4 billion entries) [S1, book].
*   **Strengths:** Avoids the excessive simplex count of recursive contour-meshing [S1, book]. Provides a direct lookup method similar to Marching Cubes but for higher dimensions [S1, book].
*   **Limitations:** Memory requirements grow doubly exponentially with dimension [S1, book]. Not practical for $n > 4$ due to table size [S1, book].
*   **When to Use:** When isosurface extraction in 4D or higher is required and memory allows [S1, book]. Not suitable for standard 3D if Marching Cubes is preferred due to simplicity, but offers an alternative geometric construction [S1, book].

## Background
The method was proposed by Bhaniramka et al. [14] to address the problem of excessive simplices in recursive contour-meshing for 4D data [S1, book]. It is described in the *Visualization Handbook* (2005) [S1, book]. The approach is analogous to the Marching Cubes algorithm but extends the concept of precomputed triangulation tables to $n$-dimensional hypercubes [S1, book].

## Sources
[S1] Time-Dependent Isosurface Extraction, HAN-WEI SHEN, in Visualization Handbook, 2005, URL: https://www.sciencedirect.com/science/article/pii/B978012387582250006X, fetched 2026-10-30 (book)
[S2] Chapter Topological Visualisation Techniques for Volume Multifield Data, URL: https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
