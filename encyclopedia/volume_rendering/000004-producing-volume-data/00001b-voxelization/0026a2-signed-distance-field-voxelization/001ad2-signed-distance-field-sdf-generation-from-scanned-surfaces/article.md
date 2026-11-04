# Signed Distance Field (SDF) Generation from Scanned Surfaces

Also called: 3D Texture Distance Field for Implicit Surfaces, Mesh SDF Generation, Mesh to SDF Voxelization, MeshDistance conversion, SDF Texture Generation from Mesh, SDF Voxelization, Signed distance to triangulated surface

## Overview
This method converts a triangulated surface mesh (typically from 3D scanning) into a 3D Signed Distance Field (SDF) texture or implicit function, where the scalar value at any grid point represents the signed Euclidean distance to the nearest surface triangle [S1][S2, unofficial][S4, unofficial]. The sign is negative inside the mesh and positive outside, determined by the closest triangle's face normal or a topological side-of-mesh test [S3, unofficial][S4, unofficial]. It differs from [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md) by evaluating the distance field directly from the explicit mesh geometry (via raycasting or spatial acceleration structures) rather than projecting 2D slices, allowing for exact analytical distances without rasterization artifacts [S2, unofficial][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangulated surface mesh in polygon (PLY) format containing vertices and triangles [S1]. The mesh must be watertight (holes closed) to ensure correct topological sign determination [S3, unofficial].
*   **Output:** A 3D texture (or grid) where each voxel stores the signed distance. In GPU implementations, this is often stored in the red channel of a fragment shader output [S2, unofficial].

**Data Structures:**
*   **Mesh:** Stored as a 1D array of vertices with a stride (e.g., 16384 bytes per vertex in some GPU implementations) [S2, unofficial].
*   **Grid/Texture:** A 3D grid of resolution $N \times N \times N$ (or $res \times res \times res$). The domain is bounded by texture coordinates in the $[0, 1]$ range, representing a $1 \times 1 \times 1$ cube centered at the origin [S3, unofficial].
*   **Parameters:**
    *   `textureBoundary`: Defines the box centered at origin bounded by $|x| \le (0.5, 0.5, 0.5)$ [S3, unofficial].
    *   `textureShell`: A small value (e.g., $0.05$) defining a boundary region around the texture where distance values might be invalid [S3, unofficial].

**Algorithm Steps:**

1.  **Pre-processing:** Close any holes in the input mesh (e.g., the 5 holes in the Stanford Bunny) to ensure the interior/exterior distinction is well-defined [S3, unofficial].
2.  **Grid Initialization:** For each grid point (voxel center) in the 3D texture:
    a.  Map the 3D grid position to a normalized coordinate system. If using a 2D UV mapping for a 3D texture (as in some GPU implementations), map UV coordinates to 3D grid coordinates: $co = texCoord \times vec2(res, res^2)$, where $res$ is the resolution [S2, unofficial].
    b.  Convert to integer grid position $pos$ in range $[0, res)$ for x and y, and $[0, res^2)$ for z [S2, unofficial].
    c.  Normalize grid position to range $[-1, 1]$: $pos = (pos / res) \times 2.0 - 1.0$ [S2, unofficial].
3.  **Distance Calculation:**
    a.  Initialize `dist` to a large value (e.g., $10000.0$) [S2, unofficial].
    b.  Iterate through all triangles in the mesh. For each triangle with vertices $a, b, c$:
        i.  Compute the Euclidean distance $d$ from the current grid point $pos$ to the triangle surface using a function like $udTriangle(pos, a, b, c)$ [S2, unofficial].
        ii. Update `dist = min(dist, d)` [S2, unofficial].
    c.  Determine the sign of the distance. This is typically done by checking the parity of ray intersections (inside/outside status) or by using the normal of the closest triangle [S1][S3, unofficial]. If the point is inside, the distance is negative; if outside, positive.
4.  **Boundary Handling (Texture Lookup):**
    When querying the SDF at runtime or finalizing the texture, handle points outside the primary texture bounds using the `textureBoundary` and `textureShell` parameters [S3, unofficial]:
    a.  Calculate initial difference: $di = |x| - textureBoundary + textureShell$ [S3, unofficial].
    b.  If point $x$ is outside the 3D texture space (determined by $di$), calculate distance as the L2 norm of the positive components of $(di + textureShell)$: $res = ||\max(di + textureShell, 0)||_2$ [S3, unofficial].
    c.  Calculate maximum component $c = \max(di_x, \max(di_y, di_z))$ [S3, unofficial].
    d.  If $c < 0$ (point is inside the texture space), adjust coordinate $x = x + textureBoundary$ and lookup the distance from the 3D texture: $res = lookupTexture(texture, x)$ [S3, unofficial].
    e.  Return $res$ [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_sdf_from_mesh(mesh, resolution):
    """
    Generates a 3D SDF texture from a triangulated mesh.
    
    Args:
        mesh: List of triangles, each triangle is a list of 3 vertices (vec3).
        resolution: Integer, the dimension of the cubic grid (N x N x N).
        
    Returns:
        A 3D array (or texture) of size (resolution, resolution, resolution)
        containing signed distance values.
    """
    # Initialize grid with a large value
    # Grid dimensions: x, y, z in [0, resolution-1]
    grid = [[[10000.0 for _ in range(resolution)] for _ in range(resolution)] for _ in range(resolution)]
    
    # Pre-process mesh: Ensure watertight (holes closed) [S3, unofficial]
    # This step is assumed to be handled by the input mesh preparation
    if not is_watertight(mesh):
        close_mesh_holes(mesh)
        
    # Iterate over every voxel in the grid
    for z in range(resolution):
        for y in range(resolution):
            for x in range(resolution):
                # Normalize grid position to [-1, 1] range [S2, unofficial]
                # Assuming the mesh is centered at origin and fits in [-0.5, 0.5]^3
                # Or mapped to [-1, 1] as per [S2, unofficial]
                pos_x = (x / resolution) * 2.0 - 1.0
                pos_y = (y / resolution) * 2.0 - 1.0
                pos_z = (z / resolution) * 2.0 - 1.0
                point = vec3(pos_x, pos_y, pos_z)
                
                min_dist = 10000.0
                closest_normal = vec3(0.0, 0.0, 0.0)
                is_inside = False
                
                # Iterate through all triangles to find minimum distance [S2, unofficial]
                for triangle in mesh:
                    a, b, c = triangle[0], triangle[1], triangle[2]
                    
                    # Compute distance from point to triangle [S2, unofficial]
                    d, normal = distance_to_triangle(point, a, b, c)
                    
                    if d < min_dist:
                        min_dist = d
                        closest_normal = normal
                
                # Determine sign based on inside/outside test [S1][S3, unofficial]
                # A common method is ray casting parity or using the closest triangle's normal
                # If the point is inside, distance is negative; otherwise positive.
                # For simplicity, we assume a ray-casting parity test here [S1]
                if is_point_inside_mesh(point, mesh):
                    is_inside = True
                
                if is_inside:
                    grid[z][y][x] = -min_dist
                else:
                    grid[z][y][x] = min_dist
                    
    return grid

def distance_to_triangle(point, a, b, c):
    """
    Computes the Euclidean distance from a point to a triangle and the normal of the closest point.
    This is a standard geometric routine [S1][S2, unofficial].
    """
    # Implementation of point-to-triangle distance
    # Returns (distance, closest_normal)
    # ... (Standard geometric calculation) ...
    pass

def is_point_inside_mesh(point, mesh):
    """
    Determines if a point is inside a mesh using ray casting parity.
    Cast a ray from the point and count intersections with triangles.
    Odd number of intersections -> Inside. Even -> Outside. [S1]
    """
    # ... (Ray casting implementation) ...
    pass
```

**Edge Cases and Pitfalls:**
*   **Holes in Mesh:** Open meshes (e.g., Stanford Bunny with 5 holes) will produce incorrect signs and distances if not closed [S3, unofficial].
*   **Boundary Values:** The `textureShell` value is not universal and must be adjusted for different textures to balance between avoiding invalid boundary data and maximizing the valid lookup area [S3, unofficial].
*   **Scanner Accuracy:** The accuracy of the SDF depends on the surface scanner and the accuracy of the signed distance computation [S1].
*   **Resolution:** Low resolution can lose detail. For complex meshes, higher resolutions (e.g., $N=128$) may be required [S1].

## Subtopics

### [Signed Distance Field from Mesh](1327-signed-distance-field-from-mesh.md)
Converts a triangulated surface mesh into a 3D SDF texture or implicit function, evaluating distance directly from explicit mesh geometry via raycasting or spatial acceleration structures, allowing for exact analytical distances without rasterization artifacts.

### [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md)
Computes an SDF by iterating over a 3D grid, determining inside/outside status via ray-casting parity on 2D slices, and sampling the exact mesh distance at each voxel center, explicitly using 2D slice intersections for topological containment.

### Shell SDF
An unsigned distance field minus a shell half-thickness. Negative within `shell_voxels` of any triangle, positive beyond. Works on open, non-manifold, self-intersecting, and inconsistently wound meshes. Extracts a thin solid around every triangle: two walls around an open sheet, and an outer surface plus an inner one for a closed solid [S4, unofficial]. Parameter `shell_voxels` is the shell half-thickness in voxels, default 1.5f, must be $\ge \sqrt{3}/2$ (0.866) to prevent holes between voxels, and must be $<$ `trunc_dist` [S4, unofficial]. This is the unsigned offset field from Xu & Barbic, 'Signed Distance Fields for Polygon Soup Meshes' (GI 2014), before their interior-component removal step [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N^3 \cdot T)$, where $N$ is the grid resolution and $T$ is the number of triangles in the mesh. This is because each voxel must be tested against all triangles (or a subset via acceleration structure) to find the minimum distance [S2, unofficial].
*   **Memory Complexity:** $O(N^3)$ for storing the 3D texture/grid [S3, unofficial].
*   **Measured Performance:** For the Stanford Bunny (35,947 vertices, 69,451 triangles) and an automobile part (156,229 vertices, 312,458 triangles), the method was applied to grids of varying resolutions [S1]. For tetrahedral grids, resolution was increased to $N=128$ to avoid losing detail [S1].
*   **Strengths:** Provides exact analytical distances to the mesh surface, avoiding rasterization artifacts associated with slice-based methods [S2, unofficial]. Can handle complex mesh geometries [S3, unofficial].
*   **Limitations:** Computationally expensive for high-resolution grids and large meshes due to the brute-force or raycasting nature of distance calculation [S2, unofficial]. Accuracy depends on the input mesh quality and scanner accuracy [S1].
*   **When to Use:** When high-quality, exact SDFs are needed for implicit surface rendering or simulation, and the mesh is not excessively large or the resolution not extremely high.
*   **When Not to Use:** For real-time applications with very large meshes and high-resolution requirements, where approximate methods or hierarchical structures (like [Dense Volume SDF Generation](4499-dense-volume-sdf-generation.md)) might be more suitable.

## Background
The method originates from the need to represent complex scanned surfaces as implicit functions for rendering and simulation. The Stanford Bunny mesh (35,947 vertices, 69,451 triangles) and an aluminum alloy A413 automobile part (156,229 vertices, 312,458 triangles) are commonly used test cases [S1]. The approach involves placing the model in a bounded 3D grid and sampling the distance values to the surface for every point in that grid, storing the raw floating-point data as a 3D distance texture [S3, unofficial]. Tools like MeshDistance from the GEL (GEometry and Linear algebra) framework have been used for this conversion [S3, unofficial]. The accuracy of the extraction depends on the surface scanner and signed distances accuracies [S1].

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S4] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
