# Modified Marching Cubes with Triangle Decimation

Also called: 3D Isosurface Generation (Marching Cubes), Marching Cubes with Decimation

## Overview
This method generates an isosurface from a 3D scalar field by applying a modified Marching Cubes algorithm that resolves topological ambiguities, followed by a triangle decimation step to reduce mesh complexity and improve visual quality [S1, unofficial]. It distinguishes itself from standard Marching Cubes by explicitly handling ambiguity cases and reducing the triangle count through vertex averaging, making it suitable for large-scale visualizations where storage and rendering performance are critical [S1, unofficial].

## How it works
**Inputs:** A 3D scalar field defined on a structured grid (e.g., FDS grid), an isovalue threshold $L$, and the grid dimensions.
**Outputs:** A triangle mesh $M = (V, T)$ with vertex positions, connectivity, and per-vertex normal vectors.

**Data Structures:**
*   **Grid:** 3D array of scalar values $T(x,y,z)$.
*   **Cube:** A unit cell aligned with the grid, defined by 8 corner nodes indexed 0 to 7.
*   **Case Index:** An 8-bit integer (0–255) representing the binary state of the cube corners relative to $L$.
*   **Mesh:** Lists of vertices $V$ and triangles $T$. Each triangle references 3 vertices.

**Steps:**

1.  **Grid Partitioning:** Split the region to be contoured into cubes, each aligned with the underlying grid [S1, unofficial].
2.  **Cube Processing:** For each cube:
    a.  **Corner Numbering:** Number the 8 corners from 0 to 7 [S1, unofficial].
    b.  **Binary Encoding:** For each corner $i$, assign bit $b_i = 1$ if $T_i > L$, else $b_i = 0$. Construct the case index $C = \sum_{i=0}^{7} b_i 2^i$ [S1, unofficial].
    c.  **Triangulation:** Use the case index $C$ to determine the triangulation pattern. Store vertex locations and edge connections [S1, unofficial]. *Note: The source specifies a modified algorithm that removes ambiguities, though the specific lookup table modifications are not detailed in the provided text [S1, unofficial].*
3.  **Triangle Decimation:** Iterate through the generated triangles to reduce count [S1, unofficial]:
    a.  Identify "small" triangles: any triangle where two or more vertices are closest to the same grid node [S1, unofficial].
    b.  For each identified triangle, compute a new vertex position $\mathbf{v}_{new}$ as the average of the three removed vertices' coordinates:
        $$ \mathbf{v}_{new} = \frac{\mathbf{v}_1 + \mathbf{v}_2 + \mathbf{v}_3}{3} $$
    c.  Replace the three vertices with $\mathbf{v}_{new}$ and re-triangulate the affected local region [S1, unofficial].
4.  **Normal Estimation:** For each vertex in the final mesh, compute a normal vector [S1, unofficial]:
    a.  Identify all triangles $n$ sharing the vertex.
    b.  For each triangle, compute the area $A_n$ and unit normal $\mathbf{U}_n$.
    c.  Compute the harmonic weighted average of the normals [S1, unofficial].

**Formulas:**

*   **Uncertainty Bound:** The uncertainty $\Delta x$ in isosurface location between two data points $(x_1, T_1)$ and $(x_2, T_2)$ is bounded by [S1, unofficial]:
    $$ \left| \frac{\Delta x}{x_2 - x_1} \right| \leq \frac{\max(|\Delta T_1|, |\Delta T_2|)}{|T_2 - T_1|} $$
    where $\Delta T_1$ and $\Delta T_2$ are the uncertainties in the data values at $x_1$ and $x_2$.

*   **Vertex Normal:** The normal $\mathbf{V}_i$ for vertex $i$ is calculated as [S1, unofficial]:
    $$ \mathbf{V}_i = \frac{\sum_n \frac{\mathbf{U}_n}{A_n}}{\sum_n \frac{1}{A_n}} = \frac{\sum_n \frac{\mathbf{u}_n \times \mathbf{v}_n}{\|\mathbf{u}_n \times \mathbf{v}_n\|^2}}{\sum_n \frac{1}{\|\mathbf{u}_n \times \mathbf{v}_n\|}} $$
    where for the $n$-th triangle, $\mathbf{u}_n$ and $\mathbf{v}_n$ are two edge vectors, $A_n = \|\mathbf{u}_n \times \mathbf{v}_n\|$ is the area, and $\mathbf{U}_n = \frac{\mathbf{u}_n \times \mathbf{v}_n}{A_n}$ is the unit normal.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modified_marching_cubes_with_decimation(grid, isovalue):
    """
    grid: 3D array of scalar values
    isovalue: L, the threshold for isosurface extraction
    Returns: vertices (V), triangles (T), normals (N)
    """
    V = []
    T = []
    
    # Step 1 & 2: Marching Cubes with Ambiguity Removal
    for each cube in grid:
        # 2a: Number corners 0-7
        corner_values = [grid[c] for c in cube_corners]
        
        # 2b: Binary encoding
        case_index = 0
        for i, val in enumerate(corner_values):
            if val > isovalue:
                case_index |= (1 << i)
        
        # 2c: Triangulate (using modified lookup table to remove ambiguities)
        # The specific ambiguity resolution logic is not detailed in the source [S1, unofficial]
        # Standard MC lookup is assumed modified here
        triangles = triangulate_cube(case_index, cube_corners, isovalue)
        
        # Add vertices and triangles to global lists
        for tri in triangles:
            T.append(tri)
            # Vertices are added if not already present (shared between cubes)
            for v_idx in tri:
                if v_idx not in V_indices:
                    V.append(cube_corners[v_idx].position)
                    V_indices.add(v_idx)

    # Step 3: Triangle Decimation
    # Identify triangles to collapse: those with 2+ vertices closest to the same grid node
    triangles_to_remove = []
    for tri in T:
        vertices = [V[v] for v in tri]
        # Determine closest grid node for each vertex
        closest_nodes = [get_closest_grid_node(v) for v in vertices]
        # Check if two or more vertices share the same closest node
        if len(set(closest_nodes)) < 3:
            triangles_to_remove.append(tri)

    # Remove triangles and create new vertex
    new_vertices = []
    for tri in triangles_to_remove:
        v1, v2, v3 = V[tri[0]], V[tri[1]], V[tri[2]]
        # Average coordinates
        new_v = (v1 + v2 + v3) / 3.0
        new_vertices.append(new_v)
        # Mark old vertices for removal or reuse
        # Re-triangulation logic is described as "re-triangulate using this new vertex" [S1, unofficial]
        # This implies local remeshing around the new vertex, details not provided [S1, unofficial]

    # Merge new vertices into V and update T
    # Note: Exact re-triangulation algorithm is not specified in the source [S1, unofficial]
    
    # Step 4: Normal Estimation
    N = []
    for i, v in enumerate(V):
        # 4a: Find triangles sharing vertex i
        adjacent_triangles = get_adjacent_triangles(i, T)
        
        sum_weighted_normal = [0.0, 0.0, 0.0]
        sum_inverse_area = 0.0
        
        for tri in adjacent_triangles:
            # 4b: Compute triangle normal and area
            p0, p1, p2 = V[tri[0]], V[tri[1]], V[tri[2]]
            u = p1 - p0
            v_vec = p2 - p0
            cross = cross_product(u, v_vec)
            area = length(cross)
            
            if area > 0:
                unit_normal = cross / area
                # 4c: Harmonic weight (1/Area)
                weight = 1.0 / area
                sum_weighted_normal += unit_normal * weight
                sum_inverse_area += weight
        
        if sum_inverse_area > 0:
            normal = sum_weighted_normal / sum_inverse_area
            N.append(normalize(normal))
        else:
            N.append([0.0, 0.0, 0.0]) # Default normal

    return V, T, N
```

**Edge Cases and Pitfalls:**
*   **Ambiguities:** The source mentions a "modified" algorithm to remove ambiguities but does not specify the modification (e.g., edge case handling for the 6 ambiguous cases in standard MC) [S1, unofficial]. Implementation requires consulting the specific ambiguity resolution strategy used in the source software (Smokeview) [S1, unofficial].
*   **Decimation Criteria:** The criterion "two or more vertices closest to the same node" requires a robust definition of "closest" (e.g., Euclidean distance to grid nodes) [S1, unofficial].
*   **Re-triangulation:** The step "re-triangulate using this new vertex" is vague. It likely involves creating new triangles connecting the new vertex to the neighbors of the collapsed region, but the exact connectivity update is not defined [S1, unofficial].
*   **Precision:** The uncertainty formula suggests that data uncertainty $\Delta T$ directly impacts spatial accuracy. High uncertainty in the input grid leads to larger $\Delta x$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly given. The base Marching Cubes is $O(N)$ where $N$ is the number of cubes. Decimation adds a pass over triangles. Normal estimation requires iterating over adjacent triangles, which can be $O(T)$ where $T$ is the number of triangles.
*   **Memory Complexity:** Stores the full mesh $V, T$ and normals $N$. Decimation reduces $|T|$, lowering memory usage compared to standard MC [S1, unofficial].
*   **Performance:** Reduces storage requirements and improves visual quality by removing small, high-aspect-ratio triangles [S1, unofficial].
*   **Strengths:** Handles ambiguities, reduces mesh complexity, provides smooth normals via harmonic averaging.
*   **Limitations:** The decimation and re-triangulation steps are not fully detailed, potentially leading to implementation ambiguity. The method is specific to the FDS grid structure mentioned in the source [S1, unofficial].

## Background
The method originates from the Smokeview visualization tool for Fire Dynamics Simulation (FDS) data. It is described in the Smokeview Technical Reference Guide, which details the implementation of isosurface generation for fire simulation data [S1, unofficial]. The technique was developed to handle the large datasets generated by FDS, balancing visual fidelity with computational efficiency [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
