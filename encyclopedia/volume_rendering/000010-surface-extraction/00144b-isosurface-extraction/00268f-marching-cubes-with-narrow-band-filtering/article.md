# Marching Cubes with Narrow-Band Filtering

Also called: MeshReconstruction::MarchCube

## Overview
This method reconstructs a triangle mesh from a Signed Distance Function (SDF) by evaluating the SDF on a regular grid of cubes, filtering out cubes that are too far from the surface (narrow-band filtering), and applying the standard Marching Cubes algorithm only to the intersected cubes [S1, unofficial]. It differs from standard Marching Cubes by skipping the vast majority of the domain volume, significantly reducing computation time for sparse or bounded surfaces [S1, unofficial].

## How it works
**Inputs:**
- `domain`: A bounding box defined by `min` (Vec3) and `size` (Vec3).
- `sdf`: A function `Fun3s` mapping `Vec3` to `double`, representing the signed distance function.
- `cubeSize`: The dimensions of the grid cells (Vec3).
- `isoLevel`: The scalar value at which to extract the surface (default 0.0).
- `sdfGrad`: An optional gradient function for vertex normals. If `nullptr`, normals are approximated numerically [S1, unofficial].

**Outputs:**
- `mesh`: A triangle mesh containing vertices and indices.

**Data Structures:**
- `Cube`: Represents a single grid cell.
  - `pos[8]`: Array of 8 vertex positions (Vec3).
  - `sdf[8]`: Array of 8 SDF values (double) at the vertices.
  - `signConfigToIntersectedEdges[256]`: Lookup table mapping vertex sign configurations (0-255) to a 12-bit bitmask of intersected edges.
  - `edges[12]`: Array of 12 edge definitions, each containing an `edgeFlag` (12-bit integer, powers of 2) and two vertex indices (`vert0`, `vert1`, 0-7).

**Parameters:**
- `NumCubes`: Integer constant, default 50. Used to calculate default `cubeSize` as `domain.size * (1.0 / NumCubes)` [S1, unofficial].

**Algorithm Steps:**
1.  **Grid Setup**: Calculate grid dimensions:
    $$ NumX = \lceil domain.size.x / cubeSize.x \rceil $$
    $$ NumY = \lceil domain.size.y / cubeSize.y \rceil $$
    $$ NumZ = \lceil domain.size.z / cubeSize.z \rceil $$
    Calculate geometric constants:
    $$ HalfCubeDiag = \|cubeSize\| / 2.0 $$
    $$ HalfCubeSize = cubeSize * 0.5 $$

2.  **Domain Traversal**: Iterate over grid indices $ix, iy, iz$ from $0$ to $NumX-1$, $NumY-1$, $NumZ-1$.

3.  **Cube Construction**:
    - Calculate cube minimum corner:
      $$ min = domain.min + \{ ix \cdot cubeSize.x, iy \cdot cubeSize.y, iz \cdot cubeSize.z \} $$
    - Calculate cube center:
      $$ cubeCenter = min + HalfCubeSize $$
    - Calculate distance to iso-surface:
      $$ dist = |sdf(cubeCenter) - isoLevel| $$

4.  **Narrow-Band Filtering**:
    - If $dist > HalfCubeDiag$, the cube is too far from the surface to intersect it. Continue to the next cube [S1, unofficial].

5.  **SDF Sampling**:
    - Construct a `Cube` object. Sample the SDF at the 8 vertices of the cube to populate `Cube.sdf[8]` and store positions in `Cube.pos[8]` [S1, unofficial].

6.  **Intersection Detection**:
    - Determine the sign configuration index $idx$ by checking the sign of each vertex's SDF value relative to `isoLevel`. If $sdf[i] \ge isoLevel$, set bit $i$ in $idx$.
    - Look up `intersectedEdgesMask = signConfigToIntersectedEdges[idx]`.
    - If `intersectedEdgesMask == 0`, no intersection; continue to next cube.

7.  **Vertex Interpolation**:
    - For each edge $e$ in $0..11$:
      - If `(intersectedEdgesMask & edges[e].edgeFlag) != 0`:
        - Interpolate the intersection point on the edge using linear interpolation:
          $$ \mu = \frac{isoLevel - sdf[edges[e].vert0]}{sdf[edges[e].vert1] - sdf[edges[e].vert0]} $$
          $$ p_{intersect} = pos[edges[e].vert0] + \mu \cdot (pos[edges[e].vert1] - pos[edges[e].vert0]) $$
        - Handle edge cases: If $|isoLevel - sdf[vert0]| < \epsilon$ or $|sdf[vert0] - sdf[vert1]| < \epsilon$, use the vertex position directly [S1, unofficial].

8.  **Triangulation**:
    - Use the standard Marching Cubes lookup table (derived from `signConfigToIntersectedEdges` and edge connectivity) to determine which triangles to form from the intersection vertices.
    - Add triangles to `mesh` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_narrow_band(domain, sdf, cube_size, iso_level=0.0, sdf_grad=None):
    # 1. Grid Setup
    num_x = ceil(domain.size.x / cube_size.x)
    num_y = ceil(domain.size.y / cube_size.y)
    num_z = ceil(domain.size.z / cube_size.z)
    
    half_cube_diag = length(cube_size) / 2.0
    half_cube_size = cube_size * 0.5
    
    mesh = Mesh()
    
    # 2. Domain Traversal
    for ix in range(num_x):
        for iy in range(num_y):
            for iz in range(num_z):
                # 3. Cube Construction
                min_corner = domain.min + Vec3(ix * cube_size.x, iy * cube_size.y, iz * cube_size.z)
                cube_center = min_corner + half_cube_size
                
                # 4. Narrow-Band Filtering
                dist = abs(sdf(cube_center) - iso_level)
                if dist > half_cube_diag:
                    continue
                
                # 5. SDF Sampling
                cube_vertices = []
                cube_sdf_values = []
                for vx in range(2):
                    for vy in range(2):
                        for vz in range(2):
                            pos = min_corner + Vec3(vx * cube_size.x, vy * cube_size.y, vz * cube_size.z)
                            cube_vertices.append(pos)
                            cube_sdf_values.append(sdf(pos))
                
                # 6. Intersection Detection
                sign_config = 0
                for i in range(8):
                    if cube_sdf_values[i] >= iso_level:
                        sign_config |= (1 << i)
                
                intersected_edges_mask = sign_config_to_intersected_edges[sign_config]
                if intersected_edges_mask == 0:
                    continue
                
                # 7. Vertex Interpolation
                intersection_points = []
                for e in range(12):
                    edge_flag = edges[e].edge_flag
                    v0_idx = edges[e].vert0
                    v1_idx = edges[e].vert1
                    
                    if intersected_edges_mask & edge_flag:
                        v0_val = cube_sdf_values[v0_idx]
                        v1_val = cube_sdf_values[v1_idx]
                        p0 = cube_vertices[v0_idx]
                        p1 = cube_vertices[v1_idx]
                        
                        eps = 1e-5
                        if abs(iso_level - v0_val) < eps:
                            intersection_points.append(p0)
                        elif abs(v0_val - v1_val) < eps:
                            intersection_points.append(p0)
                        else:
                            mu = (iso_level - v0_val) / (v1_val - v0_val)
                            p_intersect = p0 + (p1 - p0) * mu
                            intersection_points.append(p_intersect)
                
                # 8. Triangulation
                # Note: The specific triangle indices depend on the standard MC lookup table
                # associated with sign_config. This implementation assumes a helper function
                # that maps sign_config and intersection_points to triangle indices.
                triangles = get_marching_cubes_triangles(sign_config, intersection_points)
                mesh.add_triangles(triangles)
                
                # Optional: Compute normals if sdf_grad is provided
                if sdf_grad is not None:
                    for tri in triangles:
                        # Compute face normal or vertex normals
                        pass
                else:
                    # Numerical approximation or default normals
                    pass
                    
    return mesh
```

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of cubes in the narrow band. The narrow-band filtering reduces the effective $N$ from the total grid volume to only those cubes near the surface.
- **Memory Complexity**: $O(N)$ for storing the mesh and temporary cube data.
- **Performance**: Significantly faster than standard Marching Cubes for large domains with small surfaces because it skips non-intersecting cubes [S1, unofficial].
- **Strengths**: Efficiency for sparse SDFs; simplicity of implementation using precomputed lookup tables [S1, unofficial].
- **Limitations**: Requires the SDF to be evaluated at cube centers and vertices; narrow-band threshold must be chosen carefully to avoid missing thin features [S1, unofficial].

## Background
This method is implemented in the `MeshReconstruction` library by Magnus2, which uses the Marching Cubes algorithm with narrow-band filtering to efficiently reconstruct triangle meshes from SDFs [S1, unofficial]. It is designed to be self-contained, fast, and portable, requiring only C++14 [S1, unofficial].

## Sources
[S1] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
