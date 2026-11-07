# Marching Cubes with Narrow-Band Filtering

Also called: MeshReconstruction::MarchCube

## Overview
This method reconstructs a triangle mesh from a Signed Distance Function (SDF) by evaluating the field on a grid of cubes, filtering out cubes that are too far from the surface (narrow-band filtering), and applying the standard Marching Cubes algorithm to the remaining cubes [S1, unofficial]. It differs from standard Marching Cubes by skipping the vast majority of voxels that do not intersect the isosurface, significantly reducing computation time for sparse or bounded domains [S1, unofficial].

## How it works
**Inputs:**
*   `sdf`: A function or data structure returning the signed distance value for any 3D point.
*   `domain`: A bounding box (`Rect3`) defined by `min` and `size` vectors, enclosing the region of interest.
*   `cubeSize`: The dimensions of the cubic voxels. If not provided, defaults to `domain.size / 50` [S1, unofficial].
*   `isoLevel`: The scalar value at which to extract the surface (default 0) [S1, unofficial].
*   `sdfGrad`: An optional gradient function for vertex normals. If `nullptr`, normals are computed numerically [S1, unofficial].

**Data Structures:**
*   `Cube`: A class representing a single voxel.
    *   `pos[8]`: Array of 8 vertex positions in world space.
    *   `sdf[8]`: Array of 8 SDF values at the vertices.
    *   `signConfigToIntersectedEdges[256]`: Precomputed lookup table mapping the 8-bit vertex sign configuration to a 12-bit bitmask of intersected edges. The bitmask uses powers of 2 ($2^0$ to $2^{11}$) corresponding to edges $e_0$ through $e_{11}$ [S1, unofficial].
    *   `edges[12]`: Array of edge definitions, each containing a bitmask flag and the indices of the two vertices it connects (e.g., edge 0 connects vertex 0 and 1) [S1, unofficial].
*   `mesh`: The output triangle mesh.

**Algorithm Steps:**

1.  **Grid Setup**: Calculate the number of cubes along each axis:
    $$ NumX = \lceil domain.size.x / cubeSize.x \rceil $$
    $$ NumY = \lceil domain.size.y / cubeSize.y \rceil $$
    $$ NumZ = \lceil domain.size.z / cubeSize.z \rceil $$
    Calculate geometric constants:
    $$ HalfCubeSize = cubeSize \times 0.5 $$
    $$ HalfCubeDiag = \|cubeSize\| / 2.0 $$
    [S1, unofficial]

2.  **Iterate Voxels**: Loop over grid indices $ix, iy, iz$.
    *   Calculate the minimum corner of the cube:
        $$ min = domain.min + \{ix \cdot cubeSize.x, iy \cdot cubeSize.y, iz \cdot cubeSize.z\} $$
    *   Calculate the cube center:
        $$ cubeCenter = min + HalfCubeSize $$
    *   Evaluate the SDF at the center: $d_{center} = sdf(cubeCenter)$.
    *   Calculate the distance to the isosurface:
        $$ dist = |d_{center} - isoLevel| $$
    *   **Narrow-Band Filter**: If $dist > HalfCubeDiag$, the cube cannot intersect the isosurface. Continue to the next cube [S1, unofficial].

3.  **Cube Construction**: For cubes passing the filter:
    *   Initialize a `Cube` object with the domain space and SDF function.
    *   Sample the SDF at the 8 vertices of the cube to populate `Cube.sdf[8]` and store positions in `Cube.pos[8]` [S1, unofficial].

4.  **Intersection**:
    *   Determine the case index $C$ by checking the sign of each vertex's SDF value relative to `isoLevel`. If $sdf[v_i] < isoLevel$ (or $> isoLevel$, depending on convention), set bit $i$ of $C$ to 1.
    *   Look up the intersected edges using `signConfigToIntersectedEdges[C]`.
    *   For each intersected edge $k$, calculate the intersection point using linear interpolation:
        $$ \mu = \frac{isoLevel - sdf[v_{start}]}{sdf[v_{end}] - sdf[v_{start}]} $$
        $$ p_{intersect} = pos[v_{start}] + \mu \cdot (pos[v_{end}] - pos[v_{start}]) $$
        Handle edge cases where $|isoLevel - sdf[v]| < \epsilon$ or $|sdf[v_{start}] - sdf[v_{end}]| < \epsilon$ by returning the vertex position directly [S1, unofficial].

5.  **Triangulation**:
    *   Use the `triangleTable` (from parent article) to determine which triangles to form from the intersection points.
    *   Generate triangles and add them to the `mesh`.
    *   Compute vertex normals using `sdfGrad` if provided, otherwise use numerical approximation [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def march_cubes_narrow_band(sdf, domain, cube_size, iso_level=0.0, sdf_grad=None):
    # 1. Grid Setup
    num_x = ceil(domain.size.x / cube_size.x)
    num_y = ceil(domain.size.y / cube_size.y)
    num_z = ceil(domain.size.z / cube_size.z)
    
    half_cube_size = cube_size * 0.5
    half_cube_diag = norm(cube_size) / 2.0
    
    mesh = Mesh()
    
    # 2. Iterate Voxels
    for ix in range(num_x):
        for iy in range(num_y):
            for iz in range(num_z):
                # Calculate cube bounds
                min_corner = domain.min + Vector3(ix * cube_size.x, iy * cube_size.y, iz * cube_size.z)
                cube_center = min_corner + half_cube_size
                
                # Narrow-band filter
                dist_to_surface = abs(sdf(cube_center) - iso_level)
                if dist_to_surface > half_cube_diag:
                    continue
                
                # 3. Cube Construction
                cube = Cube(min_corner, cube_size, sdf)
                
                # 4. Intersection
                # Determine case index C based on signs of 8 vertices relative to iso_level
                case_index = cube.get_case_index(iso_level)
                
                # Get intersected edges bitmask
                edges_mask = sign_config_to_intersected_edges[case_index]
                
                intersection_points = []
                for edge_idx in range(12):
                    if (edges_mask >> edge_idx) & 1:
                        edge = edges[edge_idx]
                        v1_idx = edge.vert0
                        v2_idx = edge.vert1
                        
                        # Linear interpolation
                        val1 = cube.sdf[v1_idx]
                        val2 = cube.sdf[v2_idx]
                        pos1 = cube.pos[v1_idx]
                        pos2 = cube.pos[v2_idx]
                        
                        eps = 1e-5
                        if abs(iso_level - val1) < eps:
                            p = pos1
                        elif abs(iso_level - val2) < eps:
                            p = pos2
                        elif abs(val1 - val2) < eps:
                            p = pos1
                        else:
                            mu = (iso_level - val1) / (val2 - val1)
                            p = pos1 + mu * (pos2 - pos1)
                        
                        intersection_points.append(p)
                
                # 5. Triangulation
                # Use triangle_table[case_index] to form triangles from intersection_points
                # Add triangles to mesh
                triangulate(case_index, intersection_points, mesh)
                
                # Compute normals
                if sdf_grad:
                    compute_normals_with_gradient(mesh, sdf_grad)
                else:
                    compute_normals_numerical(mesh)
                    
    return mesh
```

## Subtopics

### [Snap Marching Cubes](2181-snap-marching-cubes.md)
Snap vertices to grid lines to reduce artifacts. Choose when high precision alignment is needed.

### [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md)
Export formats. Choose when saving results.

### [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md)
Parallelize the voxel loop. Choose for large domains.

### [Marching Cubes Vertex List Generation](4447-marching-cubes-vertex-list-generation.md)
Optimized vertex storage. Choose for memory-constrained environments.

### [Lookup Table and Configuration](4978-lookup-table-and-configuration.md)
LUT generation details. Choose when customizing edge/vertex ordering.

### [Topology and Ambiguity Handling](6547-topology-and-ambiguity-handling.md)
Resolve saddle point ambiguities. Choose when topological correctness is critical.

### [Marching Cubes Iso-surface Extraction](6921-marching-cubes-iso-surface-extraction.md)
Standard MC implementation. Choose when narrow-band filtering is not applicable.

### [Mesh Simplification and Optimization](8588-mesh-simplification-and-optimization.md)
Reduce polygon count post-extraction. Choose for real-time rendering.

### [Surface-following Marching Cubes with Blocking](8966-surface-following-marching-cubes-with-blocking.md)
Process only relevant blocks. Choose for extremely sparse data.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of cubes in the domain, but effectively much faster due to narrow-band filtering. The number of processed cubes is proportional to the surface area times the band width, rather than the volume.
*   **Memory Complexity**: $O(1)$ auxiliary space per cube, plus $O(F)$ for the output mesh where $F$ is the number of faces.
*   **Performance**: Significantly faster than standard Marching Cubes for SDFs with small support or large empty regions, as it avoids evaluating and processing cubes far from the surface [S1, unofficial].
*   **Strengths**: Efficiency, simplicity, self-contained implementation [S1, unofficial].
*   **Limitations**: Requires an SDF or a method to determine distance to surface for filtering. The narrow-band threshold must be chosen carefully; too small a band may miss parts of the surface if the grid is coarse, too large a band reduces performance benefits.
*   **When to use**: When extracting surfaces from SDFs or scalar fields where the surface is localized within a known bounding box.
*   **When not to use**: When the surface fills the entire domain (filtering provides no benefit) or when the distance to the surface is not available for efficient center-point evaluation.

## Background
The method is implemented in the `MeshReconstruction` library by Magnus2, which provides a self-contained C++14 implementation [S1, unofficial]. It builds upon the classic Marching Cubes algorithm [S1, unofficial] and the concept of narrow-band filtering for efficient surface extraction. The library is released under the MIT License and has been tested on Windows 10 with Visual Studio 2017, though it is portable [S1, unofficial].

## Sources
[S1] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
