# Marching Cubes with Narrow-Band Filtering

Also called: MeshReconstruction::MarchCube

## Overview
This method reconstructs a triangle mesh from a Signed Distance Function (SDF) by evaluating the field on a grid of cubes, filtering out cubes that are too far from the surface (narrow-band filtering), and applying the standard Marching Cubes algorithm to the remaining cubes [S1, unofficial]. It differs from standard Marching Cubes by skipping the vast majority of voxels that do not intersect the isosurface, significantly reducing computational cost for sparse or bounded domains [S1, unofficial].

## How it works
**Inputs:**
*   `sdf`: A function $S: \mathbb{R}^3 \to \mathbb{R}$ returning the signed distance to the surface.
*   `domain`: A bounding box defined by `min` and `size` vectors.
*   `cubeSize`: The dimensions of the cubic grid cells.
*   `isoLevel`: The scalar value at which the surface is extracted (typically 0 for SDFs) [S1, unofficial].
*   `sdfGrad`: (Optional) A function returning the gradient $\nabla S$ at a point. If null, numerical approximation is used [S1, unofficial].

**Outputs:**
*   A triangle mesh (vertices and indices) representing the isosurface [S1, unofficial].

**Data Structures:**
*   `Cube`: Represents a single voxel. Stores the 8 vertex positions (`pos[8]`) and the 8 SDF values (`sdf[8]`) in local space.
*   `signConfigToIntersectedEdges`: A lookup table of 256 integers. Each integer is a 12-bit bitmask where bit $k$ corresponds to edge $k$ (standard numbering: $e_0..e_{11}$). If bit $k$ is set, the isosurface intersects edge $k$ [S1, unofficial].
*   `edges`: An array of 12 `Edge` structs. Each struct contains:
    *   `edgeFlag`: A power-of-two integer ($2^k$) corresponding to the bit position in the lookup table.
    *   `vert0`, `vert1`: The indices (0–7) of the two vertices defining the edge [S1, unofficial].

**Algorithm Steps:**

1.  **Grid Setup:**
    Calculate the number of cubes along each axis:
    $$
    \text{NumX} = \lceil \frac{\text{domain.size.x}}{\text{cubeSize.x}} \rceil, \quad
    \text{NumY} = \lceil \frac{\text{domain.size.y}}{\text{cubeSize.y}} \rceil, \quad
    \text{NumZ} = \lceil \frac{\text{domain.size.z}}{\text{cubeSize.z}} \rceil
    $$
    Calculate geometric constants:
    $$
    \text{HalfCubeDiag} = \frac{\|\text{cubeSize}\|}{2.0}, \quad
    \text{HalfCubeSize} = \text{cubeSize} \times 0.5
    $$
    [S1, unofficial]

2.  **Iterate and Filter:**
    Loop over grid indices $i_x \in [0, \text{NumX})$, $i_y \in [0, \text{NumY})$, $i_z \in [0, \text{NumZ})$.
    *   Calculate the minimum corner of the cube in world space:
        $$
        \text{min} = \text{domain.min} + (i_x \cdot \text{cubeSize.x}, i_y \cdot \text{cubeSize.y}, i_z \cdot \text{cubeSize.z})
        $$
    *   Calculate the cube center:
        $$
        \text{cubeCenter} = \text{min} + \text{HalfCubeSize}
        $$
    *   Evaluate the SDF at the center and compute the distance to the iso-level:
        $$
        \text{dist} = |S(\text{cubeCenter}) - \text{isoLevel}|
        $$
    *   **Narrow-Band Filter:** If $\text{dist} > \text{HalfCubeDiag}$, the cube is too far from the surface to intersect it. Skip to the next cube [S1, unofficial].

3.  **Cube Construction:**
    For cubes passing the filter, construct the `Cube` object:
    *   Store the 8 vertex positions relative to the domain space.
    *   Evaluate $S$ at each of the 8 vertices and store in `Cube.sdf[8]` [S1, unofficial].

4.  **Intersection Detection:**
    Call `cube.Intersect(isoLevel)`:
    *   Compute the case index $C$ by checking the sign of each vertex's SDF value relative to `isoLevel`. If $S(v_i) < \text{isoLevel}$ (or $> \text{isoLevel}$, depending on convention), set bit $i$ of $C$.
    *   Look up `signConfigToIntersectedEdges[C]` to get the bitmask of intersected edges.
    *   For each set bit $k$, find the intersection point on edge $k$ using linear interpolation between the two vertices of that edge:
        $$
        \mu = \frac{\text{isoLevel} - S(v_{\text{start}})}{S(v_{\text{end}}) - S(v_{\text{start}})}
        $$
        $$
        P_{\text{intersect}} = v_{\text{start}} + \mu (v_{\text{end}} - v_{\text{start}})
        $$
        *   Handle edge cases: If $|S(v_{\text{start}}) - \text{isoLevel}| < \epsilon$, return $v_{\text{start}}$. If $|S(v_{\text{end}}) - S(v_{\text{start}})| < \epsilon$, return $v_{\text{start}}$ [S1, unofficial].

5.  **Triangulation:**
    Call `Triangulate(intersect, sdfGrad, mesh)`:
    *   Use the standard Marching Cubes triangle connection table (from parent article) to determine which intersection points form triangles based on the case index $C$.
    *   Add the generated triangles to the output mesh.
    *   If `sdfGrad` is provided, use it for vertex normals; otherwise, compute normals numerically [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def marching_cubes_narrow_band(sdf, domain_min, domain_size, cube_size, iso_level=0.0, sdf_grad=None):
    """
    Reconstruct mesh from SDF using narrow-band filtering.
    """
    # 1. Grid Setup
    num_x = math.ceil(domain_size.x / cube_size.x)
    num_y = math.ceil(domain_size.y / cube_size.y)
    num_z = math.ceil(domain_size.z / cube_size.z)
    
    half_cube_diag = math.sqrt(cube_size.x**2 + cube_size.y**2 + cube_size.z**2) / 2.0
    half_cube_size = (cube_size.x * 0.5, cube_size.y * 0.5, cube_size.z * 0.5)
    
    mesh_vertices = []
    mesh_triangles = []
    
    # Precomputed lookup table from parent article (signConfigToIntersectedEdges)
    # And edge definitions (edges array)
    # For brevity, assume these are available as global constants or passed in.
    # signConfigToIntersectedEdges: list of 256 ints
    # edges: list of 12 dicts with 'flag', 'v0', 'v1'
    
    for ix in range(num_x):
        for iy in range(num_y):
            for iz in range(num_z):
                # 2. Cube Position
                min_corner = (
                    domain_min.x + ix * cube_size.x,
                    domain_min.y + iy * cube_size.y,
                    domain_min.z + iz * cube_size.z
                )
                cube_center = (
                    min_corner[0] + half_cube_size[0],
                    min_corner[1] + half_cube_size[1],
                    min_corner[2] + half_cube_size[2]
                )
                
                # 3. Narrow-Band Filter
                dist_to_surface = abs(sdf(cube_center) - iso_level)
                if dist_to_surface > half_cube_diag:
                    continue
                
                # 4. Construct Cube and Evaluate SDF at Vertices
                # Vertices 0-7 relative to min_corner
                # Standard ordering: 0=(0,0,0), 1=(1,0,0), 2=(1,1,0), 3=(0,1,0), 
                #                    4=(0,0,1), 5=(1,0,1), 6=(1,1,1), 7=(0,1,1)
                vertex_positions = []
                vertex_sdfs = []
                
                for x_bit in [0, 1]:
                    for y_bit in [0, 1]:
                        for z_bit in [0, 1]:
                            pos = (
                                min_corner[0] + x_bit * cube_size.x,
                                min_corner[1] + y_bit * cube_size.y,
                                min_corner[2] + z_bit * cube_size.z
                            )
                            vertex_positions.append(pos)
                            vertex_sdfs.append(sdf(pos))
                
                # 5. Intersection Detection
                # Calculate Case Index C
                case_index = 0
                for i in range(8):
                    # Convention: bit set if SDF < iso_level (or >, must match LUT)
                    # Assuming standard: if sdf < iso, bit is 1
                    if vertex_sdfs[i] < iso_level:
                        case_index |= (1 << i)
                
                intersected_edges_mask = signConfigToIntersectedEdges[case_index]
                
                intersection_points = []
                for edge_idx, edge_def in enumerate(edges):
                    if intersected_edges_mask & (1 << edge_idx):
                        v0_idx = edge_def['v0']
                        v1_idx = edge_def['v1']
                        
                        p0 = vertex_positions[v0_idx]
                        p1 = vertex_positions[v1_idx]
                        s0 = vertex_sdfs[v0_idx]
                        s1 = vertex_sdfs[v1_idx]
                        
                        # Linear Interpolation
                        eps = 1e-5
                        if abs(s0 - iso_level) < eps:
                            pt = p0
                        elif abs(s1 - s0) < eps:
                            pt = p0
                        else:
                            mu = (iso_level - s0) / (s1 - s0)
                            pt = (
                                p0[0] + mu * (p1[0] - p0[0]),
                                p0[1] + mu * (p1[1] - p0[1]),
                                p0[2] + mu * (p1[2] - p0[2])
                            )
                        intersection_points.append(pt)
                
                # 6. Triangulation
                # Use triangleTable from parent article
                # triangleTable[case_index] returns list of edge indices forming triangles, terminated by -1
                tri_indices = triangleTable[case_index]
                
                i = 0
                while tri_indices[i] != -1:
                    e0 = tri_indices[i]
                    e1 = tri_indices[i+1]
                    e2 = tri_indices[i+2]
                    i += 3
                    
                    # Add vertices to mesh
                    # Note: In a real implementation, vertices are often shared. 
                    # Here we assume simple list append for clarity, 
                    # but proper implementation uses a vertex buffer/index buffer.
                    mesh_vertices.append(intersection_points[e0])
                    mesh_vertices.append(intersection_points[e1])
                    mesh_vertices.append(intersection_points[e2])
                    
                    # Add triangle indices (assuming sequential vertex addition)
                    base_idx = len(mesh_vertices) - 3
                    mesh_triangles.append((base_idx, base_idx+1, base_idx+2))
                    
    return mesh_vertices, mesh_triangles
```

**Parameters:**
*   `NumCubes`: Integer constant, default 50. Used to derive default `cubeSize` as `domain.size / NumCubes` [S1, unofficial].
*   `isoLevel`: Default 0. Changing this moves the reconstructed surface [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Narrow-Band Threshold:** If `dist > HalfCubeDiag`, the cube is skipped. This is the primary optimization but requires careful tuning if the SDF is noisy or the grid is coarse [S1, unofficial].
*   **Ambiguity:** Cases 46, 47, 98, 99, 210, 213, 214, 215 contain saddle points. The standard LUT uses a fixed triangulation, which can cause cracks if adjacent voxels resolve ambiguities differently [S7, S8, unofficial]. This method does not inherently resolve these; topology handling must be addressed separately or via parent article's ambiguity handling [S7, S8, unofficial].
*   **Precision:** Linear interpolation assumes the SDF is well-behaved between vertices. If `sdf(v1) == sdf(v2)`, the method returns `v1` to avoid division by zero [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of cubes in the domain. However, the constant factor is significantly reduced by the narrow-band filter, which skips cubes far from the surface. The actual work is proportional to the number of intersected cubes [S1, unofficial].
*   **Memory Complexity:** $O(V)$ where $V$ is the number of vertices in the output mesh. The grid storage is $O(N)$ but only if the entire grid is stored; typically, cubes are processed on-the-fly, so memory is dominated by the output mesh and the small lookup tables [S1, unofficial].
*   **Performance:** Fast due to precomputed lookup tables and narrow-band filtering which excludes cubes far from the surface [S1, unofficial].
*   **Strengths:** Efficient for sparse or bounded domains; simple implementation; leverages standard Marching Cubes topology.
*   **Limitations:** Relies on the SDF being accurate; narrow-band filtering may miss thin features if the grid is too coarse; does not resolve topological ambiguities inherently [S1, S7, S8, unofficial].
*   **When to use:** When reconstructing surfaces from SDFs where the domain is known and bounded, and performance is critical.
*   **When not to use:** When the domain is unbounded or the SDF is not available; when topological correctness in ambiguous cases is critical without additional processing [S1, S7, S8, unofficial].

## Background
This method is implemented in the `MeshReconstruction` C++ library by Magnus2 [S1, unofficial]. It is designed to be self-contained with no dependencies, requiring C++14, and tested on Visual Studio 2017 and Windows 10 [S1, unofficial]. It is licensed under the MIT License [S1, unofficial]. The approach combines the classic Marching Cubes algorithm with a spatial filtering heuristic to optimize performance for SDF-based reconstruction [S1, unofficial].

## Sources
[S1] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
[S7] Topology and Ambiguity Handling, 6547-topology-and-ambiguity-handling.md
[S8] Marching Cubes Iso-surface Extraction, 6921-marching-cubes-iso-surface-extraction.md
