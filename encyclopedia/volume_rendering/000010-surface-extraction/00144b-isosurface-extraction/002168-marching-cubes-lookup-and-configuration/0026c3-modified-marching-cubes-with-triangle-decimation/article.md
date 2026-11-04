# Modified Marching Cubes with Triangle Decimation

Also called: 3D Isosurface Generation (Marching Cubes), Marching Cubes with Decimation

## Overview
This method generates an isosurface from 3D scalar field data by applying a modified marching cubes algorithm that resolves topological ambiguities and subsequently reduces polygon count via triangle decimation [S1, unofficial]. It distinguishes itself from standard marching cubes by explicitly handling ambiguous cases to ensure manifold correctness and by post-processing the mesh to collapse degenerate triangles, thereby reducing storage requirements and improving visual quality [S1, unofficial].

## How it works
**Inputs:** A 3D scalar field defined on a structured grid (e.g., FDS grid), a threshold value $L$, and the grid geometry.
**Outputs:** A triangle mesh (vertices, indices) and associated vertex normals.

### Data Structures
*   **Grid:** Structured 3D array of scalar values $T(x,y,z)$.
*   **Cube:** Defined by 8 corner nodes. Corners are numbered 0 to 7. The specific numbering scheme is referenced in the source figures [S1, unofficial].
*   **Case Index:** An 8-bit integer (0–255) derived from the binary state of the cube corners.
*   **Triangle List:** Stores vertex coordinates and edge indices for each generated triangle.
*   **Vertex Normal:** A 3D vector stored per unique vertex.

### Algorithm Steps

1.  **Grid Partitioning:** Split the region to be contoured into a number of cubes, each aligned with the underlying grid [S1, unofficial].
2.  **Cube Evaluation:** For each cube:
    a.  Number cube corners from 0 to 7 [S1, unofficial].
    b.  For each corner $i$ (beginning with 0), assign a bit $b_i = 1$ if the scalar value $T_i$ exceeds the threshold $L$, otherwise $b_i = 0$ [S1, unofficial].
    c.  Compute the case number $C = \sum_{i=0}^{7} b_i 2^i$. This yields an integer between 0 and 255 [S1, unofficial].
    d.  Triangulate the cube surface based on case $C$. Store vertex locations and edge numbers according to the determined case. The source notes that the case numbers are determined using a base-3 or binary mapping, with specific examples given for rows of the lookup table (e.g., first row: 0, 40, 1, 2, 3) [S1, unofficial]. *Note: The source mentions "modified to remove ambiguities," implying a specific lookup configuration or tie-breaking rule is applied for ambiguous cases (e.g., cases 1, 5, 9, 29, 41, 45, 61, 65, 69, 77, 85, 89, 93, 101, 109, 117, 125, 129, 133, 141, 149, 157, 165, 173, 181, 189, 197, 205, 213, 221, 229, 237, 245, 253), though the exact tie-breaking logic is not detailed in the text [S1, unofficial].*
3.  **Triangle Decimation:** Eliminate small triangles to reduce polygon count [S1, unofficial].
    a.  Identify triangles where two or more vertices are closest to the same grid node.
    b.  Replace the removed triangle vertices with a single new vertex. The coordinates of this new vertex are the average of the coordinates of the three removed vertices.
    c.  Re-triangulate the local geometry using this new vertex [S1, unofficial].
4.  **Normal Estimation:** Estimate normal directions for each vertex to support smooth shading [S1, unofficial].
    a.  Determine all triangles sharing the vertex.
    b.  Compute the normal vector for each triangle.
    c.  Construct a harmonic weighted average of the triangle normal vectors. The weight for each triangle is inversely proportional to its area.
    d.  Store the resulting average normal with the vertex data [S1, unofficial].

### Formulas

**Uncertainty Bound:**
The uncertainty in isosurface location $\Delta x$ between two node data value pairs $(x_1, T_1)$ and $(x_2, T_2)$ is bounded by [S1, unofficial]:

$$
\left| \frac{\Delta x}{x_2 - x_1} \right| \leq \frac{\max(|\Delta T_1|, |\Delta T_2|)}{|T_2 - T_1|}
$$

where $\Delta T_1$ and $\Delta T_2$ are the uncertainties in $T_1$ and $T_2$ at $x_1$ and $x_2$ respectively.

**Vertex Normal Calculation:**
The averaged normal vector $V_i$ for vertex $i$ is determined by summing over all triangles $n$ connected to vertex $i$ [S1, unofficial]:

$$
V_i = \frac{\sum_n \frac{U_n}{A_n}}{\sum_n \frac{1}{A_n}} = \frac{\sum_n \frac{u_n \times v_n}{\|u_n \times v_n\|^2}}{\sum_n \frac{1}{\|u_n \times v_n\|}}
$$

where for the $n$-th triangle:
*   $u_n$ and $v_n$ are two side vectors of the triangle.
*   $A_n = \|u_n \times v_n\|$ is the triangle area.
*   $U_n = \frac{u_n \times v_n}{A_n}$ is the unit vector normal to the triangle.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modified_marching_cubes_with_decimation(grid, threshold):
    """
    Generates an isosurface from a 3D grid using modified marching cubes
    and triangle decimation.
    
    Args:
        grid: 3D array of scalar values.
        threshold: The isovalue L.
        
    Returns:
        vertices: List of vertex coordinates.
        normals: List of vertex normal vectors.
        triangles: List of triangle indices.
    """
    all_triangles = []
    vertex_map = {} # Map from (x,y,z) to vertex index for deduplication
    
    # Step 1 & 2: Process each cube
    for x in range(grid.width - 1):
        for y in range(grid.height - 1):
            for z in range(grid.depth - 1):
                # Extract cube corners (0-7)
                # Corner numbering assumed to be standard binary indexing
                # 0: (x,y,z), 1: (x+1,y,z), 2: (x+1,y+1,z), 3: (x,y+1,z)
                # 4: (x,y,z+1), 5: (x+1,y,z+1), 6: (x+1,y+1,z+1), 7: (x,y+1,z+1)
                corners = [
                    grid[x, y, z],
                    grid[x+1, y, z],
                    grid[x+1, y+1, z],
                    grid[x, y+1, z],
                    grid[x, y, z+1],
                    grid[x+1, y, z+1],
                    grid[x+1, y+1, z+1],
                    grid[x, y+1, z+1]
                ]
                
                # Step 2b: Assign bits
                case_index = 0
                for i, val in enumerate(corners):
                    if val > threshold:
                        case_index |= (1 << i)
                
                # Step 2d: Triangulate
                # Note: The specific edge lookup table and ambiguity resolution
                # are implementation details not fully specified in the source.
                # This step calls a helper that returns triangle indices relative to cube vertices.
                cube_triangles = triangulate_cube(case_index, x, y, z, grid)
                all_triangles.extend(cube_triangles)
    
    # Step 3: Triangle Decimation
    # The source describes eliminating triangles where two or more vertices
    # are closest to the same node. This requires a spatial index or
    # proximity check.
    simplified_triangles = decimate_triangles(all_triangles, grid)
    
    # Step 4: Normal Estimation
    # Build adjacency list
    vertex_triangles = build_vertex_triangle_adjacency(simplified_triangles)
    
    vertices = []
    normals = []
    
    for v_idx, tri_indices in vertex_triangles.items():
        # Get vertex position
        pos = get_vertex_position(v_idx, simplified_triangles)
        
        # Compute harmonic weighted average normal
        sum_weighted_normal = [0.0, 0.0, 0.0]
        sum_inverse_area = 0.0
        
        for t_idx in tri_indices:
            tri_verts = get_triangle_vertices(t_idx, simplified_triangles)
            u = vector_sub(tri_verts[1], tri_verts[0])
            v = vector_sub(tri_verts[2], tri_verts[0])
            cross = vector_cross(u, v)
            area = vector_length(cross)
            
            if area > 0:
                # U_n = cross / area
                # Weight = 1 / area
                # Contribution = U_n / area = cross / area^2
                inv_area_sq = 1.0 / (area * area)
                sum_weighted_normal[0] += cross[0] * inv_area_sq
                sum_weighted_normal[1] += cross[1] * inv_area_sq
                sum_weighted_normal[2] += cross[2] * inv_area_sq
                sum_inverse_area += 1.0 / area
        
        if sum_inverse_area > 0:
            normal = [
                sum_weighted_normal[0] / sum_inverse_area,
                sum_weighted_normal[1] / sum_inverse_area,
                sum_weighted_normal[2] / sum_inverse_area
            ]
            # Normalize the final result
            normal = normalize(normal)
        else:
            normal = [0.0, 0.0, 1.0] # Default fallback
            
        vertices.append(pos)
        normals.append(normal)
        
    return vertices, normals, simplified_triangles

def decimate_triangles(triangles, grid):
    """
    Eliminates small triangles where two or more vertices are closest 
    to the same grid node. Replaces removed vertices with their average.
    """
    # Implementation detail: The source does not provide the exact algorithm
    # for identifying "closest to the same node" or the re-triangulation logic.
    # This is a placeholder for the decimation step described in [S1, unofficial].
    # [G] In practice, this involves clustering vertices by their nearest grid cell
    # and collapsing clusters.
    return triangles # Placeholder
```

### Edge Cases and Pitfalls
*   **Ambiguities:** Standard marching cubes has ambiguous cases (e.g., saddle points). The source states the algorithm is "modified to remove ambiguities" but does not specify the exact tie-breaking rule or lookup table variant used [S1, unofficial]. Implementers must choose a consistent ambiguity resolution strategy (e.g., Lucas's method or specific NIST recommendations).
*   **Decimation Logic:** The decimation step ("eliminate small triangles... replace removed triangle vertices with a vertex with coordinates that are the average") is geometrically complex. Re-triangulating after vertex collapse requires care to maintain manifold connectivity. The source does not detail the re-triangulation algorithm [S1, unofficial].
*   **Precision:** The uncertainty formula depends on the data variation $|T_2 - T_1|$. If $T_1 \approx T_2$, the bound becomes large, indicating high uncertainty in the isosurface location [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of grid cells, as each cell is processed once. Decimation adds overhead proportional to the number of triangles.
*   **Memory Complexity:** Proportional to the number of triangles generated. Decimation reduces this compared to standard marching cubes.
*   **Strengths:** Reduces storage requirements and improves visual quality by removing degenerate geometry. Handles 3D isosurfaces effectively.
*   **Limitations:** The decimation step is not fully specified in the source, making implementation non-trivial. Ambiguity resolution requires careful configuration.
*   **When to use:** When visual quality and storage efficiency are priorities for 3D scalar field visualization, particularly in fire dynamics simulation data (FDS) [S1, unofficial].

## Background
The method originates from the Smokeview tool for visualizing Fire Dynamics Simulation (FDS) data. It is described in the "Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide" [S1, unofficial]. The technique builds on the classic marching cubes algorithm [15] (referenced in the source) but adds modifications for ambiguity removal and triangle decimation to handle the specific requirements of fire simulation visualization [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
