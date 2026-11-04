# Extended Marching Cubes with No Degenerate Triangles

Also called: Extended Isosurface Lookup Table Method

## Overview
Extended Marching Cubes is a variant of the Marching Cubes algorithm that utilizes a preprocessed lookup table to generate isosurfaces free of degenerate (zero-area) triangles [S1, unofficial]. It ensures that the resulting isosurface strictly separates regions of positive and negative scalar values relative to the isovalue [S1, unofficial]. Unlike standard Marching Cubes, which may produce degenerate geometry in specific edge cases, this method guarantees non-collinear vertices for all generated triangles [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D scalar grid field $f$, an isovalue $\sigma$, and the dimensions of the grid.
*   **Output:** A set of non-degenerate triangles representing the isosurface $f^{-1}(\sigma)$.

**Data Structures**
*   **Lookup Table:** A preprocessed table stored in XML format. Each entry corresponds to a vertex labeling configuration (combinations of '+', '-', '=') and contains a list of triangle definitions. The table is preprocessed to exclude any degenerate triangles [S1, unofficial].
*   **Vertex Labels:** Each grid vertex $v$ is assigned a label based on its scalar value $s_v$ relative to $\sigma$:
    *   '+' if $s_v > \sigma$
    *   '-' if $s_v < \sigma$
    *   '=' if $s_v = \sigma$ [S1, unofficial]

**Algorithm Steps**
1.  **Label Grid Vertices:** Iterate through all grid vertices and assign labels ('+', '-', '=') based on the comparison of their scalar values with the isovalue $\sigma$ [S1, unofficial].
2.  **Cube Processing:** For each grid cube:
    a. Determine the vertex labeling configuration of the cube's 8 corners.
    b. Retrieve the corresponding triangle list from the Extended Lookup Table [S1, unofficial].
    c. For each triangle in the list, identify the required vertices. These vertices are defined as either:
        *   A cube vertex with label '='.
        *   The midpoint of a "bipolar" edge (an edge connecting a '+' vertex and a '-' vertex) [S1, unofficial].
    d. **Reposition Vertices:** For vertices defined as midpoints of bipolar edges, reposition them using linear interpolation to their exact intersection with the isosurface. The position $p$ on an edge between vertices $v_1$ (value $s_1$) and $v_2$ (value $s_2$) is calculated as:
        $$ p = v_1 + \frac{\sigma - s_1}{s_2 - s_1} (v_2 - v_1) $$
        Vertices defined as '=' remain at their grid positions [S1, unofficial].
    e. Add the resulting triangles to the output mesh [S1, unofficial].
3.  **Boundary Processing:** After processing all internal cubes, handle the grid boundaries to ensure completeness:
    a. **Four '=' Vertices:** For every square facet on the grid boundary where all four vertices are labeled '=', use a diagonal to form two isosurface triangles covering that square [S1, unofficial].
    b. **Three '=' Vertices:** For every square facet on the grid boundary where three vertices are labeled '=' and one is labeled '-', form a single isosurface triangle from the three vertices labeled '=' [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extended_marching_cubes(grid, isovalue, lookup_table):
    """
    grid: 3D array of scalar values
    isovalue: float sigma
    lookup_table: dict mapping label_tuple -> list of triangle_indices
    Returns: list of triangles (each triangle is a list of 3 vertices)
    """
    triangles = []
    rows, cols, depth = grid.shape
    
    # Step 1: Label grid vertices
    # Labels: 1 for '+', -1 for '-', 0 for '='
    labels = np.zeros((rows, cols, depth), dtype=int)
    for i in range(rows):
        for j in range(cols):
            for k in range(depth):
                val = grid[i, j, k]
                if val > isovalue:
                    labels[i, j, k] = 1
                elif val < isovalue:
                    labels[i, j, k] = -1
                else:
                    labels[i, j, k] = 0

    # Helper for linear interpolation on bipolar edges
    def interpolate(v1, s1, v2, s2):
        # v1, v2 are coordinates; s1, s2 are scalar values
        # Assumes s1 != s2 (bipolar edge)
        t = (isovalue - s1) / (s2 - s1)
        return v1 + t * (v2 - v1)

    # Step 2: Process internal cubes
    for i in range(rows - 1):
        for j in range(cols - 1):
            for k in range(depth - 1):
                # Get labels for the 8 corners of the cube
                # Order: typically binary index 0-7
                cube_labels = [
                    labels[i, j, k], labels[i+1, j, k], labels[i+1, j+1, k], labels[i, j+1, k],
                    labels[i, j, k+1], labels[i+1, j, k+1], labels[i+1, j+1, k+1], labels[i, j+1, k+1]
                ]
                
                # Map labels to lookup key (e.g., tuple of 1, -1, 0)
                label_key = tuple(cube_labels)
                
                # Retrieve triangle definitions
                # Each definition specifies which vertices to use
                # Vertices are either grid vertices (if label == 0) or edge midpoints
                tri_defs = lookup_table.get(label_key, [])
                
                for tri_def in tri_defs:
                    tri_vertices = []
                    for vertex_spec in tri_def:
                        if vertex_spec.type == 'GRID_VERTEX':
                            # Vertex is a grid corner with label '='
                            idx = vertex_spec.index
                            tri_vertices.append((i + (idx & 1), j + ((idx >> 1) & 1), k + ((idx >> 2) & 1)))
                        elif vertex_spec.type == 'EDGE_MIDPOINT':
                            # Vertex is on a bipolar edge
                            edge_indices = vertex_spec.edge_indices
                            v1_idx, v2_idx = edge_indices
                            v1_coords = get_grid_coords(v1_idx, i, j, k)
                            v2_coords = get_grid_coords(v2_idx, i, j, k)
                            s1 = grid[v1_coords]
                            s2 = grid[v2_coords]
                            tri_vertices.append(interpolate(v1_coords, s1, v2_coords, s2))
                    triangles.append(tri_vertices)

    # Step 3: Process boundaries
    # Handle faces on the boundary of the grid
    # Example: Bottom face (k=0)
    for i in range(rows - 1):
        for j in range(cols - 1):
            # Get labels for the 4 corners of the boundary square
            l00 = labels[i, j, 0]
            l10 = labels[i+1, j, 0]
            l11 = labels[i+1, j+1, 0]
            l01 = labels[i, j+1, 0]
            
            count_eq = sum(1 for l in [l00, l10, l11, l01] if l == 0)
            count_neg = sum(1 for l in [l00, l10, l11, l01] if l == -1)
            
            if count_eq == 4:
                # Four '=' vertices: use diagonal to form two triangles
                # Vertices are grid points
                v00 = (i, j, 0)
                v10 = (i+1, j, 0)
                v11 = (i+1, j+1, 0)
                v01 = (i, j+1, 0)
                triangles.append([v00, v10, v11])
                triangles.append([v00, v11, v01])
            elif count_eq == 3 and count_neg == 1:
                # Three '=' vertices: form triangle from '=' vertices
                eq_vertices = []
                if l00 == 0: eq_vertices.append((i, j, 0))
                if l10 == 0: eq_vertices.append((i+1, j, 0))
                if l11 == 0: eq_vertices.append((i+1, j+1, 0))
                if l01 == 0: eq_vertices.append((i, j+1, 0))
                triangles.append(eq_vertices)
                
    return triangles
```

**Parameters**
*   **Isovalue ($\sigma$):** The scalar threshold used to classify vertices. Typical values depend on the dataset (e.g., 100 for aneurism data, 30 for bonsai data) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Degenerate Triangles:** The extended lookup table is preprocessed to ensure no degenerate triangles exist. The algorithm never positions two triangle vertices at the same location or moves three vertices onto the same grid edge, ensuring no collinear vertices [S1, unofficial].
*   **Duplicate Triangles:** If three or four vertices labeled '=' lie on a grid facet surrounded by '-' vertices, identical but oppositely oriented triangles are created in incident cubes. This causes the isosurface to collapse onto itself [S1, unofficial].
*   **Non-Manifold Surfaces:** When a grid vertex scalar value equals the isovalue, the trilinear interpolant's critical points may cause the isosurface to not be a manifold [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of grid cubes and the average number of triangles per cube configuration.
*   **Memory Complexity:** Linear with respect to the number of output triangles. The lookup table size is constant (256 entries, though many are unused or symmetric).
*   **Measured Performance:** On a 256x256x256 aneurism dataset, Extended MC produced 174,300 triangles in 0.73s, compared to 175,832 triangles in 0.71s for Regular MC [S1, unofficial]. On a 256x256x256 bonsai dataset, Extended MC produced 1,117,304 triangles in 1.19s, compared to 1,284,542 triangles in 1.16s for Regular MC [S1, unofficial].
*   **Strengths:** Guarantees no degenerate triangles; ensures separation of positive and negative regions.
*   **Limitations:** May produce duplicate triangles with opposite orientation in specific boundary configurations; may result in non-manifold surfaces when grid vertices equal the isovalue.

## Background
Extended Marching Cubes was introduced by S. Raman and R. Wenger. The method addresses the issue of degenerate triangles in standard Marching Cubes by using an extended lookup table. The lookup table is preprocessed and stored in XML format, with source code available for reading the table [S1, unofficial]. The method is discussed in the context of quality isosurface mesh generation, alongside techniques like SnapMC [S1, unofficial].

## Sources
[S1] Extended Marching Cubes, http://www.cse.ohio-state.edu/~wenger/publications/isomesh.pdf, fetched 2026-10-29 (unofficial)
