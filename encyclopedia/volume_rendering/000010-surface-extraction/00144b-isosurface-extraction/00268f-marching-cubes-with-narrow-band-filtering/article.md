# Marching Cubes with Narrow-Band Filtering

Also called: MeshReconstruction::MarchCube

## Overview
This method reconstructs a triangle mesh from a Signed Distance Function (SDF) by restricting the Marching Cubes algorithm to a "narrow band" of voxels near the zero-level set [S1, unofficial]. It solves the performance bottleneck of evaluating the SDF and processing geometry for the entire domain by skipping cubes that are geometrically too far from the surface [S1, unofficial]. Unlike standard Marching Cubes which processes every voxel, this approach achieves significant speedups by culling non-intersecting cubes based on the distance from the cube center to the isosurface [S1, unofficial].

## How it works
**Inputs:**
*   `sdf`: A function or data structure evaluating the signed distance to the surface at any 3D point.
*   `domain`: A bounding box (`Rect3`) defined by `min` and `size` vectors, enclosing the region of interest.
*   `isoLevel`: The scalar value defining the surface (typically 0 for SDFs) [S1, unofficial].
*   `cubeSize`: The side length of the cubic voxels. If not provided, it is derived from `domain.size` and a constant `NumCubes` (default 50) [S1, unofficial].
*   `sdfGrad`: An optional gradient function for vertex normals. If `nullptr`, normals are computed numerically [S1, unofficial].

**Outputs:**
*   A triangle mesh (vertices and indices) representing the isosurface.

**Data Structures:**
The method relies on the standard Marching Cubes Lookup Tables (LUTs) defined in [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md):
1.  `edgeTable`: 256-entry array of 12-bit bitmasks indicating which of the 12 cube edges are intersected.
2.  `triangleTable`: 256x16 array mapping case indices to triangle vertex edge indices, terminated by `-1`.

**Algorithm Steps:**

1.  **Grid Setup**:
    Calculate the grid dimensions based on the domain and cube size:
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

2.  **Voxel Iteration**:
    Iterate over all grid indices $i_x, i_y, i_z$ from $0$ to $\text{NumX}-1$, etc.

3.  **Cube Geometry**:
    For each voxel, calculate the minimum corner position in world space:
    $$
    \text{min} = \text{domain.min} + (i_x \cdot \text{cubeSize.x}, i_y \cdot \text{cubeSize.y}, i_z \cdot \text{cubeSize.z})
    $$
    Calculate the cube center:
    $$
    \text{cubeCenter} = \text{min} + \text{HalfCubeSize}
    $$

4.  **Narrow-Band Filtering (Culling)**:
    Evaluate the SDF at the cube center: $d = \text{sdf}(\text{cubeCenter})$.
    Calculate the distance to the isosurface:
    $$
    \text{dist} = |d - \text{isoLevel}|
    $$
    If $\text{dist} > \text{HalfCubeDiag}$, the cube is too far from the surface to intersect it. Skip this cube and continue to the next iteration [S1, unofficial].

5.  **Vertex Sampling**:
    For the remaining cubes, sample the SDF at the 8 vertices of the cube. Let $v_i$ be the position of vertex $i$ ($i=0..7$) and $V_i = \text{sdf}(v_i)$.

6.  **Case Index Calculation**:
    Compute the 8-bit case index $C$. For each vertex $i$, if $V_i < \text{isoLevel}$ (or $> \text{isoLevel}$, depending on the specific LUT convention used), set bit $i$ of $C$ to 1. The bit assignment must match the vertex ordering assumed by the LUT generation [S1, unofficial].

7.  **Edge Intersection**:
    Look up `edgeTable[C]` to determine which edges are intersected. For each intersected edge $k$ (connecting vertices $v_a$ and $v_b$), compute the intersection point using linear interpolation:
    $$
    \mu = \frac{\text{isoLevel} - V_a}{V_b - V_a}
    $$
    $$
    P_{\text{intersect}} = v_a + \mu (v_b - v_a)
    $$
    *Precision Guard*: If $|V_a - \text{isoLevel}| < \epsilon$ or $|V_b - \text{isoLevel}| < \epsilon$ (where $\epsilon = 10^{-5}$), return the vertex position directly to avoid division by zero or numerical instability [S1, unofficial]. If $|V_a - V_b| < \epsilon$, return $v_a$ [S1, unofficial].

8.  **Triangulation**:
    Look up `triangleTable[C]`. Read triplets of edge indices $(e_a, e_b, e_c)$ until the terminator `-1` is reached. For each triplet, retrieve the pre-computed intersection points on edges $e_a, e_b, e_c$ and add the triangle to the mesh.

9.  **Normal Computation**:
    If `sdfGrad` is provided, evaluate the gradient at each intersection point to get vertex normals. Otherwise, compute normals numerically (e.g., central differences) [S1, unofficial].

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
    
    vertices = []
    indices = []
    
    # Standard MC LUTs (edgeTable, triangleTable) assumed loaded
    # edgeTable: list of 256 ints
    # triangleTable: list of 256 lists of ints
    
    # 2. Voxel Iteration
    for ix in range(num_x):
        for iy in range(num_y):
            for iz in range(num_z):
                # 3. Cube Geometry
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
                
                # 4. Narrow-Band Filtering
                dist_to_surface = abs(sdf(cube_center) - iso_level)
                if dist_to_surface > half_cube_diag:
                    continue
                
                # 5. Vertex Sampling
                # Define 8 vertices relative to min_corner
                # Order: 0:(0,0,0), 1:(1,0,0), 2:(1,1,0), 3:(0,1,0), 
                #        4:(0,0,1), 5:(1,0,1), 6:(1,1,1), 7:(0,1,1)
                # Note: This ordering must match the LUT generation.
                # The source uses a specific bit mapping. We assume standard 
                # binary counting order for vertices 0-7 based on x,y,z bits.
                # However, the source code snippet implies a specific mapping.
                # We will define vertices explicitly.
                
                vx = cube_size.x
                vy = cube_size.y
                vz = cube_size.z
                
                v_pos = [
                    (min_corner[0], min_corner[1], min_corner[2]),       # 0
                    (min_corner[0]+vx, min_corner[1], min_corner[2]),   # 1
                    (min_corner[0]+vx, min_corner[1]+vy, min_corner[2]),# 2
                    (min_corner[0], min_corner[1]+vy, min_corner[2]),   # 3
                    (min_corner[0], min_corner[1], min_corner[2]+vz),   # 4
                    (min_corner[0]+vx, min_corner[1], min_corner[2]+vz),# 5
                    (min_corner[0]+vx, min_corner[1]+vy, min_corner[2]+vz),# 6
                    (min_corner[0], min_corner[1]+vy, min_corner[2]+vz)  # 7
                ]
                
                v_vals = [sdf(p) for p in v_pos]
                
                # 6. Case Index Calculation
                # Assuming bit i is set if v_vals[i] < iso_level (or >, check convention)
                # The source code uses a specific LUT. We assume the standard 
                # convention where bit i corresponds to vertex i.
                case_index = 0
                for i in range(8):
                    if v_vals[i] < iso_level: # Convention check needed
                        case_index |= (1 << i)
                    
                # 7. Edge Intersection
                edge_mask = edgeTable[case_index]
                edge_intersections = {}
                
                # Edge definitions: (bit_flag, vert_index_1, vert_index_2)
                # Based on source: edges[0] has flag 1, verts 0,1
                edges_def = [
                    (1, 0, 1), (2, 1, 2), (4, 2, 3), (8, 3, 0),
                    (16, 4, 5), (32, 5, 6), (64, 6, 7), (128, 7, 4),
                    (256, 0, 4), (512, 1, 5), (1024, 2, 6), (2048, 3, 7)
                ]
                
                for flag, v1_idx, v2_idx in edges_def:
                    if edge_mask & flag:
                        p1 = v_pos[v1_idx]
                        p2 = v_pos[v2_idx]
                        val1 = v_vals[v1_idx]
                        val2 = v_vals[v2_idx]
                        
                        eps = 1e-5
                        if abs(iso_level - val1) < eps:
                            interp_point = p1
                        elif abs(iso_level - val2) < eps:
                            interp_point = p2
                        elif abs(val1 - val2) < eps:
                            interp_point = p1
                        else:
                            mu = (iso_level - val1) / (val2 - val1)
                            interp_point = (
                                p1[0] + mu * (p2[0] - p1[0]),
                                p1[1] + mu * (p2[1] - p1[1]),
                                p1[2] + mu * (p2[2] - p1[2])
                            )
                        edge_intersections[flag] = interp_point
                
                # 8. Triangulation
                tri_list = triangleTable[case_index]
                i = 0
                while i < len(tri_list) and tri_list[i] != -1:
                    e1 = tri_list[i]
                    e2 = tri_list[i+1]
                    e3 = tri_list[i+2]
                    i += 3
                    
                    # Get points from edge_intersections
                    # Note: The triangle table stores edge indices (flags)
                    p_a = edge_intersections[e1]
                    p_b = edge_intersections[e2]
                    p_c = edge_intersections[e3]
                    
                    # Add to mesh
                    # In a real implementation, vertices are stored in a list
                    # and indices refer to that list. Here we assume a simple
                    # append for demonstration, but proper deduplication is needed
                    # for shared vertices. The source library likely handles this
                    # internally via a vertex list.
                    vertices.append(p_a)
                    vertices.append(p_b)
                    vertices.append(p_c)
                    
                    # Calculate normals
                    if sdf_grad:
                        n_a = sdf_grad(p_a)
                        n_b = sdf_grad(p_b)
                        n_c = sdf_grad(p_c)
                    else:
                        # Numerical approximation placeholder [G]
                        n_a = (0,0,0)
                        n_b = (0,0,0)
                        n_c = (0,0,0)
                        
                    # Store normals if the mesh structure supports it
                    # For this pseudocode, we just return vertices/indices
                    # assuming a flat list of triangles for simplicity
                    indices.append(len(vertices)-3)
                    indices.append(len(vertices)-2)
                    indices.append(len(vertices)-1)

    return vertices, indices
```

**Edge Cases and Pitfalls:**
*   **Precision**: When the isovalue is very close to a vertex value, linear interpolation can suffer from division by zero or instability. The method uses an epsilon ($10^{-5}$) to return the vertex position directly in these cases [S1, unofficial].
*   **Boundary Conditions**: The domain bounding box must fully enclose the narrow band. If the surface exits the domain, the mesh will be clipped [S1, unofficial].
*   **Ambiguity**: Cases 46, 47, 98, 99, 210, 213, 214, 215 contain topological ambiguities. The standard LUT resolves these with a fixed triangulation, which may cause cracks if adjacent voxels are processed inconsistently. This method does not inherently resolve ambiguities beyond the standard LUT [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N_{\text{band}})$, where $N_{\text{band}}$ is the number of cubes in the narrow band. This is significantly faster than $O(N_{\text{total}})$ for large domains with sparse surfaces.
*   **Memory Complexity**: $O(N_{\text{band}})$ for storing the output mesh. The LUTs are constant size.
*   **Performance**: The narrow-band filtering excludes cubes far from the surface, reducing the number of SDF evaluations and geometric operations [S1, unofficial].
*   **Strengths**: Fast, simple, self-contained, no external dependencies [S1, unofficial].
*   **Limitations**: Requires the SDF to be available. The narrow-band width is fixed by the cube size; if the cube size is too large, the band may miss thin features or the culling threshold may be too aggressive.

## Background
The method is implemented in the `MeshReconstruction` library by Magnus2 [S1, unofficial]. It combines the classic Marching Cubes algorithm [S1, unofficial] with a narrow-band optimization strategy. The library is released under the MIT License and requires C++14 [S1, unofficial].

## Sources
[S1] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
