# Modified Marching Cubes with Triangle Decimation

Also called: 3D Isosurface Generation (Marching Cubes), Marching Cubes with Decimation

## Overview
This method generates an isosurface from a 3D scalar field by applying a modified Marching Cubes algorithm to resolve topological ambiguities, followed by a triangle decimation step to reduce mesh complexity. It is distinguished from standard Marching Cubes by its explicit handling of ambiguous cases and its post-processing decimation, which collapses triangles with large aspect ratios to improve visual quality and reduce storage requirements [S1, unofficial].

## How it works
**Inputs:** A 3D scalar field $F$ defined on a structured grid (e.g., FDS grid), and an isovalue threshold $L$.
**Outputs:** A triangle mesh (vertices, indices) and associated vertex normals.

### Data Structures
*   **Grid:** Structured 3D array of scalar values.
*   **Cube:** A voxel defined by 8 corners. Corners are indexed 0 to 7. The specific numbering scheme is referenced as "Fig. 3.8" in the source [S1, unofficial].
*   **Case Index:** An 8-bit integer (0–255) derived from the binary state of the cube corners.
*   **Triangle List:** List of triangles, each defined by 3 vertex indices.
*   **Vertex Data:** Position $(x, y, z)$ and normal vector $(n_x, n_y, n_z)$.

### Algorithm Steps

1.  **Grid Partitioning:** Split the region to be contoured into cubes, each aligned with the underlying grid [S1, unofficial].
2.  **Cube Classification:** For each cube:
    a.  Number corners 0 to 7.
    b.  For each corner $i$, assign bit $b_i = 1$ if $F(\text{corner}_i) > L$, else $b_i = 0$.
    c.  Compute case index $C = \sum_{i=0}^{7} b_i 2^i$.
    d.  **Ambiguity Resolution:** The source states the algorithm is "modified to remove ambiguities" [S1, unofficial]. It references a base-3 number system mapping (0–80) to determine the case number, with specific mappings provided for certain rows (e.g., first row: 0, 40, 1, 2, 3; second row: 5, 8, 10, 11, 13; last row: 14, 16, 20) [S1, unofficial]. *Note: The exact lookup table or logic for the base-3 conversion is not fully detailed in the text, but the modification is essential for correct topology.*
    e.  Triangulate the cube surface based on the resolved case, storing vertex locations and edge numbers.
3.  **Triangle Decimation:**
    a.  Identify "small" triangles: any triangle where two or more vertices are closest to the same grid node.
    b.  Remove these triangles.
    c.  Create a new vertex at the average coordinates of the three removed vertices.
    d.  Re-triangulate the local geometry using this new vertex [S1, unofficial].
4.  **Normal Estimation:** For each vertex in the resulting mesh:
    a.  Identify all triangles sharing the vertex.
    b.  Compute the unit normal for each triangle.
    c.  Compute a harmonic weighted average of these normals, where weights are inversely proportional to triangle area.
    d.  Store the resulting normal with the vertex data [S1, unofficial].

### Formulas

**Uncertainty Bound:**
The uncertainty $\Delta x$ in the isosurface location between two node data pairs $(x_1, T_1)$ and $(x_2, T_2)$ is bounded by:
$$
\left| \frac{\Delta x}{x_2 - x_1} \right| \leq \frac{\max(|\Delta T_1|, |\Delta T_2|)}{|T_2 - T_1|}
$$
where $\Delta T_1$ and $\Delta T_2$ are the uncertainties in $T_1$ and $T_2$ respectively [S1, unofficial].

**Vertex Normal:**
The averaged normal vector $V_i$ for vertex $i$ is:
$$
V_i = \frac{\sum_n \frac{U_n}{A_n}}{\sum_n \frac{1}{A_n}} = \frac{\sum_n \frac{u_n \times v_n}{\|u_n \times v_n\|^2}}{\sum_n \frac{1}{\|u_n \times v_n\|}}
$$
where for the $n$-th triangle, $u_n$ and $v_n$ are two side vectors, $A_n = \|u_n \times v_n\|$ is the triangle area, and $U_n = \frac{u_n \times v_n}{A_n}$ is the unit normal [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modified_marching_cubes_decimation(grid, isovalue):
    vertices = []
    triangles = []
    
    # Step 1 & 2: Marching Cubes with Ambiguity Resolution
    for each cube in grid:
        # Number corners 0-7 (specific layout per Fig 3.8 in source)
        corner_values = [grid[c] for c in cube_corners]
        
        # Determine binary state
        case_bits = []
        for val in corner_values:
            case_bits.append(1 if val > isovalue else 0)
        
        case_index = sum(b << i for i, b in enumerate(case_bits))
        
        # Resolve ambiguity using modified lookup (base-3 mapping details omitted as per source limitation)
        resolved_case = resolve_ambiguity(case_index, corner_values)
        
        # Triangulate
        cube_triangles = triangulate_cube(resolved_case, cube_corners, corner_values)
        triangles.extend(cube_triangles)
        
    # Step 3: Triangle Decimation
    # Identify triangles where 2+ vertices are closest to the same grid node
    # This requires a spatial index or grid lookup to determine "closest node"
    small_triangles = []
    for tri in triangles:
        if is_small_triangle(tri, grid_nodes):
            small_triangles.append(tri)
            
    # Remove small triangles and replace with averaged vertex
    new_vertices = []
    for tri in small_triangles:
        avg_pos = average(tri.vertices)
        new_v = add_vertex(avg_pos)
        new_vertices.append(new_v)
        
    # Re-triangulate locally (specific connectivity logic not detailed in source)
    # [G] Implementation note: The source mentions "Re-triangulate using this new vertex"
    # but does not specify the exact connectivity algorithm. A standard approach would be
    # to create new triangles connecting the new vertex to the remaining boundary edges
    # of the removed region.
    
    # Step 4: Normal Estimation
    vertex_normals = {}
    for v in all_vertices:
        connected_tris = get_triangles_sharing_vertex(v)
        if not connected_tris:
            vertex_normals[v] = (0, 0, 0)
            continue
            
        numerator = (0, 0, 0)
        denominator = 0.0
        
        for tri in connected_tris:
            # Calculate area and normal
            u = tri.v2.pos - tri.v1.pos
            v_vec = tri.v3.pos - tri.v1.pos
            cross = cross_product(u, v_vec)
            area = length(cross)
            
            if area > 0:
                unit_normal = cross / area
                weight = 1.0 / area
                
                numerator += unit_normal * weight
                denominator += weight
                
        if denominator > 0:
            vertex_normals[v] = normalize(numerator / denominator)
        else:
            vertex_normals[v] = (0, 0, 0)
            
    return vertices, triangles, vertex_normals

def is_small_triangle(tri, grid_nodes):
    # Check if two or more vertices are closest to the same grid node
    closest_nodes = [find_closest_grid_node(v.pos) for v in tri.vertices]
    return len(set(closest_nodes)) < 3
```

### Parameters and Edge Cases
*   **Isovalue $L$:** Must be within the range of the scalar field.
*   **Ambiguity Resolution:** The specific base-3 mapping is critical. Failure to resolve ambiguities correctly leads to topological errors (e.g., non-manifold edges).
*   **Decimation Criteria:** The definition of "closest to the same node" depends on the grid resolution. If the grid is coarse, decimation may remove significant geometric detail.
*   **Precision:** The uncertainty formula suggests that high gradients ($|T_2 - T_1|$ large) reduce relative uncertainty, while high data noise ($\Delta T$) increases it.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for the marching cubes step, where $N$ is the number of voxels. Decimation requires spatial queries to identify "small" triangles, potentially $O(N \log N)$ or $O(N)$ with a grid hash. Normal estimation is proportional to the number of triangles.
*   **Memory Complexity:** Proportional to the number of triangles in the output mesh. Decimation reduces this compared to standard Marching Cubes.
*   **Strengths:** Reduces storage requirements and improves visual quality by removing high-aspect-ratio triangles. Handles topological ambiguities.
*   **Limitations:** The decimation step may alter the exact isosurface location, though the uncertainty is bounded. The specific ambiguity resolution logic is not fully detailed in the source, making implementation dependent on the specific base-3 mapping.
*   **When to use:** When visual fidelity and storage efficiency are prioritized over exact geometric precision, particularly in fire dynamics simulation visualization (Smokeview) [S1, unofficial].

## Background
The method originates from the Smokeview tool for visualizing Fire Dynamics Simulation (FDS) data. It was developed to provide efficient and visually accurate isosurface rendering for fire simulation results. The modification to Marching Cubes addresses topological ambiguities inherent in the standard algorithm, while the decimation step optimizes the resulting mesh for real-time rendering and storage [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
