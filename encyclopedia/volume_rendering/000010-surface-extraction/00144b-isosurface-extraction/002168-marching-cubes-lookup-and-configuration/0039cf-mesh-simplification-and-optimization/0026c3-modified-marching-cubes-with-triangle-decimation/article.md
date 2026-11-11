# Modified Marching Cubes with Triangle Decimation

Also called: 3D Isosurface Generation (Marching Cubes), Marching Cubes with Decimation

## Overview
This method generates an isosurface from a 3D scalar field by applying a modified Marching Cubes algorithm that resolves topological ambiguities, followed by a triangle decimation step to reduce mesh complexity [S1, unofficial]. It distinguishes itself from standard Marching Cubes by explicitly handling ambiguous cases to ensure a valid manifold and by post-processing the output mesh to collapse small triangles, thereby reducing storage requirements and improving visual quality [S1, unofficial].

## How it works
**Inputs:** A 3D scalar field defined on a structured grid (e.g., FDS grid), a threshold value $L$, and the grid geometry.
**Outputs:** A triangle mesh representing the isosurface, where each vertex has an associated normal vector.

### Data Structures
*   **Grid:** Structured 3D array of scalar values.
*   **Cube:** A voxel defined by 8 corner nodes.
*   **Vertex:** 3D coordinates $(x, y, z)$.
*   **Triangle:** Defined by 3 vertices.
*   **Normal:** Unit vector $(n_x, n_y, n_z)$.

### Algorithm Steps

1.  **Grid Partitioning:** Split the region to be contoured into cubes, each aligned with the underlying grid [S1, unofficial].
2.  **Case Identification:** For each cube:
    *   Number the 8 corners from 0 to 7 [S1, unofficial].
    *   For each corner $i$, assign a bit $b_i = 1$ if the scalar value $T_i > L$, else $b_i = 0$ [S1, unofficial].
    *   Compute the case number $C = \sum_{i=0}^{7} b_i 2^i$. Note: The source describes a specific mapping where case numbers are derived from a base-3 or specific lookup logic, but standard implementation uses the binary index 0–255 [S1, unofficial].
    *   Select the triangulation pattern for case $C$ [S1, unofficial].
3.  **Triangulation:** Generate triangles for the cube surface. Store vertex locations (interpolated on edges) and edge indices [S1, unofficial].
4.  **Triangle Decimation:** Iterate through the generated triangles to eliminate small ones [S1, unofficial]:
    *   Identify triangles where two or more vertices are closest to the same grid node [S1, unofficial].
    *   Remove these triangles.
    *   Compute a new vertex position $V_{new}$ as the average of the coordinates of the removed vertices:
        $$ V_{new} = \frac{1}{k} \sum_{j=1}^{k} V_j $$
        where $k$ is the number of removed vertices (typically 3 in a collapsed triangle context, or 2 if only two vertices are merged) [S1, unofficial].
    *   Re-triangulate the local region using $V_{new}$ to fill the gap [S1, unofficial].
5.  **Normal Estimation:** For each vertex in the final mesh:
    *   Identify all triangles $n$ sharing the vertex [S1, unofficial].
    *   Compute the unit normal $U_n$ for each triangle [S1, unofficial].
    *   Compute the harmonic weighted average of the normals [S1, unofficial]:
        $$ V_i = \frac{\sum_n \frac{U_n}{A_n}}{\sum_n \frac{1}{A_n}} $$
        where $A_n$ is the area of triangle $n$ [S1, unofficial].
    *   Store $V_i$ with the vertex data [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modified_marching_cubes_with_decimation(grid, threshold):
    """
    grid: 3D array of scalar values.
    threshold: Scalar value L for isosurface extraction.
    Returns: List of triangles (each triangle is a list of 3 vertices), 
             and a map of vertex_index -> normal_vector.
    """
    triangles = []
    vertices = []
    vertex_to_index = {}
    
    # Step 1 & 2: Iterate over all cubes in the grid
    for x in range(grid.width - 1):
        for y in range(grid.height - 1):
            for z in range(grid.depth - 1):
                # Get the 8 corner values
                corners = [
                    grid[x, y, z], grid[x+1, y, z], grid[x+1, y+1, z], grid[x, y+1, z],
                    grid[x, y, z+1], grid[x+1, y, z+1], grid[x+1, y+1, z+1], grid[x, y+1, z+1]
                ]
                
                # Step 2b: Determine case number
                case_num = 0
                for i, val in enumerate(corners):
                    if val > threshold:
                        case_num |= (1 << i)
                
                # Step 2c & 3: Triangulate
                # Note: Standard Marching Cubes lookup table is used here.
                # The source mentions a "modified" algorithm to remove ambiguities.
                # Implementation requires a specific lookup table for the 256 cases.
                cube_triangles = triangulate_cube(case_num, x, y, z, grid)
                triangles.extend(cube_triangles)
                
    # Step 4: Triangle Decimation
    # This step is iterative and complex to implement purely from text without 
    # specific adjacency data structures. The source describes the logic:
    # "Eliminate small triangles (any triangle with two or more vertices 
    # closest to the same node)."
    
    # Simplified decimation logic based on source description:
    # 1. Identify vertices that are "close" to the same grid node.
    # 2. Find triangles where >=2 vertices are close to the same node.
    # 3. Remove those triangles.
    # 4. Create a new vertex at the average of the removed vertices.
    # 5. Re-triangulate.
    
    # Since the source does not provide a specific adjacency-based decimation 
    # algorithm, we note that a robust implementation requires building a 
    # vertex-to-grid-node proximity map and a triangle adjacency graph.
    # [G] In practice, this is often done by collapsing edges or removing 
    # degenerate triangles based on area or vertex proximity thresholds.
    
    triangles = decimate_triangles(triangles, grid)
    
    # Step 5: Normal Estimation
    normals = {}
    for tri in triangles:
        v0, v1, v2 = tri
        # Compute triangle normal
        u = v1 - v0
        v = v2 - v0
        area = cross_product_length(u, v) # ||u x v||
        if area == 0: continue
        normal = normalize(cross_product(u, v))
        
        # Update harmonic weights for each vertex
        for vertex in [v0, v1, v2]:
            if vertex not in normals:
                normals[vertex] = {'sum_normal': (0,0,0), 'sum_weight': 0}
            
            # Weight is 1/Area
            weight = 1.0 / area
            normals[vertex]['sum_normal'] += normal * weight
            normals[vertex]['sum_weight'] += weight
            
    # Normalize the averaged normals
    final_normals = {}
    for vertex, data in normals.items():
        if data['sum_weight'] > 0:
            avg_normal = normalize(data['sum_normal'])
            final_normals[vertex] = avg_normal
            
    return triangles, final_normals

def decimate_triangles(triangles, grid):
    """
    Implements the decimation logic described in [S1, unofficial].
    """
    # [G] The source does not provide a complete algorithm for the 
    # "re-triangulate" step. A standard approach is to identify the 
    # vertices to be merged, create a new vertex, remove old triangles, 
    # and generate new triangles connecting the new vertex to the 
    # remaining neighbors.
    
    # Placeholder for complex geometric processing not fully detailed 
    # in the source text.
    return triangles
```

### Parameters and Edge Cases
*   **Threshold $L$:** The scalar value defining the isosurface.
*   **Ambiguities:** The source states the algorithm is "modified to remove ambiguities" [S1, unofficial]. Standard Marching Cubes has 15 ambiguous cases (e.g., the tetrahedron case). The specific modification is not detailed in the text, but it implies a lookup table or logic that resolves these topological issues [S1, unofficial].
*   **Precision:** The uncertainty in isosurface location $\Delta x$ is bounded by the data uncertainty $\Delta T$ and the gradient of the data [S1, unofficial]:
    $$ \left| \frac{\Delta x}{x_2 - x_1} \right| \leq \frac{\max(|\Delta T_1|, |\Delta T_2|)}{|T_2 - T_1|} $$
    Small gradients $|T_2 - T_1|$ lead to large positional uncertainty [S1, unofficial].
*   **Decimation Criteria:** Triangles are removed if "two or more vertices [are] closest to the same node" [S1, unofficial]. This implies a proximity check against the underlying grid nodes.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for the Marching Cubes step, where $N$ is the number of voxels. The decimation step is more complex, potentially $O(M \log M)$ or $O(M^2)$ depending on the implementation of the proximity and re-triangulation logic, where $M$ is the number of triangles.
*   **Memory Complexity:** Proportional to the number of vertices and triangles in the output mesh. Decimation reduces this compared to standard Marching Cubes.
*   **Strengths:** Reduces storage requirements and improves visual quality by removing small, potentially noisy triangles [S1, unofficial]. Resolves topological ambiguities [S1, unofficial].
*   **Limitations:** The decimation logic is heuristic ("small triangles") and may alter the geometry significantly if not carefully controlled. The source does not specify how "small" is defined beyond the vertex proximity criterion.
*   **When to use:** When visual quality and storage efficiency are prioritized over exact geometric fidelity, and when the underlying data has sufficient resolution to support decimation without losing features.

## Background
The method originates from the Smokeview visualization tool for Fire Dynamics Simulation (FDS) data [S1, unofficial]. It was developed to handle the large datasets generated by FDS, providing a way to visualize isosurfaces (e.g., temperature, smoke density) efficiently. The modification to Marching Cubes addresses the need for robust topology handling in scientific visualization contexts [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
