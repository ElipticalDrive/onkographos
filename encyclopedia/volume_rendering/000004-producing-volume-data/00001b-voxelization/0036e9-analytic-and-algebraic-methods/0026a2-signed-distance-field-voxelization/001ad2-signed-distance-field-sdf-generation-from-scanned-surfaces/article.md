# Signed Distance Field (SDF) Generation from Scanned Surfaces

Also called: 3D Texture Distance Field for Implicit Surfaces, Mesh SDF Generation, Mesh to SDF Voxelization, MeshDistance conversion, SDF Texture Generation from Mesh, SDF Voxelization, Signed distance to triangulated surface

## Overview
This method generates a voxelized Signed Distance Field (SDF) from a scanned triangulated surface mesh by computing the minimum Euclidean distance from each grid point to the mesh geometry [S1][S3, unofficial]. It differs from [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md) by evaluating distance directly against geometric primitives rather than projecting slices [S3, unofficial]. Unlike [Dense Volume SDF Generation](4499-dense-volume-sdf-generation.md) which may use hierarchical structures, this approach typically samples a uniform grid directly against the raw mesh triangles [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangulated surface mesh, typically in PLY format, containing vertices and triangles [S1]. The mesh must be closed (holes filled) to ensure correct signed distance representation [S3, unofficial].
*   **Output:** A 3D texture or grid where each voxel stores the minimum Euclidean distance to the nearest point on the mesh surface [S3, unofficial]. The sign is negative inside the surface and positive outside [S1]. In GPU implementations, this is often stored in the red channel of a fragment shader output [S2, unofficial].

**Data Structures:**
*   **Mesh:** Stored as a 1D array of vertices with a stride (e.g., 16384 bytes per vertex in some GPU implementations) [S2, unofficial].
*   **Grid/Texture:** A 3D texture with resolution $N \times N \times N$ (or derived from 2D UV mapping as $res \times res \times res^2$) [S2, S3, unofficial]. The domain is bounded by texture coordinates in the $[0, 1]$ range, representing a $1 \times 1 \times 1$ cube [S3, unofficial].
*   **Parameters:**
    *   `textureBoundary`: Defines the box centered at origin bounded by $|x| \le (0.5, 0.5, 0.5)$ [S3, unofficial].
    *   `textureShell`: A small value (e.g., $0.05$) defining a boundary region around the texture where distance values might be invalid [S3, unofficial].

**Algorithm Steps:**

1.  **Pre-processing:** Close any holes in the input mesh to ensure topological correctness for signed distance calculation [S3, unofficial].
2.  **Grid Initialization:** For each voxel in the 3D grid, determine its 3D position $\mathbf{x}$ in the normalized space $[-1, 1]$ or $[0, 1]$ depending on the implementation [S2, S3, unofficial].
3.  **Distance Calculation:** Compute the signed distance $\phi(\mathbf{x})$ for each grid point [S1].
    *   **Boundary Handling:** If the point $\mathbf{x}$ is outside the defined 3D texture space, calculate the distance to the texture boundary using the `textureShell` [S3, unofficial]:
        $$ d_i = |\mathbf{x}| - \text{textureBoundary} + \text{textureShell} $$
        $$ \text{res} = \| \max(d_i + \text{textureShell}, 0) \|_2 $$
    *   **Interior Lookup:** If the point is inside the texture space (specifically, if the maximum component $c = \max(d_{i_x}, \max(d_{i_y}, d_{i_z})) < 0$), adjust the coordinate and lookup the pre-computed distance from the texture [S3, unofficial]:
        $$ \mathbf{x}' = \mathbf{x} + \text{textureBoundary} $$
        $$ \text{res} = \text{lookupTexture}(\text{texture}, \mathbf{x}') $$
    *   **Sign & Magnitude:** The sign is negative inside the surface and positive outside [S1]. The magnitude is computed by iterating through all triangles and finding the closest point on each triangle to $\mathbf{x}$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_sdf_from_mesh(vertices, triangles, resolution):
    # Pre-process mesh: close holes to ensure correct signed distance representation [S3, unofficial]
    # (Mesh repair step required; not detailed in sources)
    closed_vertices = vertices
    closed_triangles = triangles

    N = resolution
    grid = [[0.0] * N for _ in range(N * N)]

    texture_boundary = 0.5
    texture_shell = 0.05

    for z in range(N):
        for y in range(N):
            for x in range(N):
                # Map grid index to 3D position in [-1, 1] range [S2, unofficial]
                pos = [
                    (x / N) * 2.0 - 1.0,
                    (y / N) * 2.0 - 1.0,
                    (z / N) * 2.0 - 1.0
                ]

                # Boundary check [S3, unofficial]
                di = [abs(pos[i]) - texture_boundary + texture_shell for i in range(3)]
                c = max(di)

                if c >= 0:
                    # Point is outside 3D texture space [S3, unofficial]
                    res = math.sqrt(sum(max(di[i] + texture_shell, 0.0) ** 2 for i in range(3)))
                else:
                    # Point is inside 3D texture space [S3, unofficial]
                    x_adj = [pos[i] + texture_boundary for i in range(3)]
                    # Normalize to [0, 1] for texture lookup
                    x_norm = [x_adj[i] for i in range(3)]
                    # Lookup pre-computed distance from 3D texture [S3, unofficial]
                    res = lookup_3d_texture(x_norm)

                grid[z * N + y][x] = res

    return grid

def lookup_3d_texture(coords):
    # Standard 3D texture lookup returning pre-computed distance value [S3, unofficial]
    # Sources do not specify the exact texture storage format or interpolation method.
    return 0.0

def compute_signed_distance(point, vertices, triangles):
    # Compute minimum Euclidean distance to mesh surface [S2, unofficial]
    min_dist = float('inf')
    for tri in triangles:
        v0, v1, v2 = vertices[tri[0]], vertices[tri[1]], vertices[tri[2]]
        d = point_to_triangle(point, v0, v1, v2)
        if d < min_dist:
            min_dist = d
    # Sign is negative inside, positive outside [S1]
    # Determined by point-in-mesh test or closest triangle normal [G]
    return -min_dist if is_inside_mesh(point, vertices, triangles) else min_dist

def point_to_triangle(p, v0, v1, v2):
    # Standard Euclidean distance from point to triangle [G]
    # Projects p onto triangle plane, checks barycentric coordinates,
    # returns distance to triangle interior or closest edge/vertex.
    return 0.0 # Standard geometric routine; implementation omitted for brevity.

def is_inside_mesh(point, vertices, triangles):
    # Standard ray-casting or winding number algorithm to determine topological containment [G]
    return False # Standard geometric routine; implementation omitted for brevity.
```

**Parameters:**
*   **Resolution:** For tetrahedral grids, resolution was increased to $N=128$ to avoid losing detail [S1].
*   **textureShell:** A small value (e.g., $0.05$) deduced by trial and error to define a boundary region around the texture where distance values might be invalid [S3, unofficial]. It should be as small as possible to maximize the valid lookup region [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes:** The mesh must be closed; holes in the mesh (e.g., the 5 holes in the Stanford Bunny) lead to incorrect distance representation [S3, unofficial].
*   **Boundary Values:** The `textureShell` value is not universal and must be adjusted for different textures to balance between avoiding invalid boundary data and maximizing the valid lookup area [S3, unofficial].
*   **Accuracy:** The accuracy of the extraction depends on the surface scanner and the accuracy of the signed distance computation [S1].

## Subtopics

### [Signed Distance Field from Mesh](1327-signed-distance-field-from-mesh.md)
This method converts an explicit triangle mesh into a voxelized Signed Distance Field (SDF) where each voxel stores the signed Euclidean distance to the nearest point on the mesh surface. The sign is negative inside the mesh and positive outside. It differs from [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md) by evaluating distance directly against geometric primitives rather than projecting slices.

### [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md)
Slice-based SDF Voxelization computes a Signed Distance Field (SDF) for a triangulated surface mesh by iterating over a 3D grid, determining inside/outside status via ray-casting parity on 2D slices, and sampling the exact mesh distance at each voxel center. It differs from [Signed Distance Field from Mesh](1327-signed-distance-field-from-mesh.md) by explicitly using 2D slice intersections to determine topological containment rather than relying on global distance field propagation or hierarchical structures for sign determination.

### Shell SDF
An unsigned distance field minus a shell half-thickness [S4, unofficial]. Negative within `shell_voxels` of any triangle, positive beyond [S4, unofficial]. Works on open, non-manifold, self-intersecting, and inconsistently wound meshes [S4, unofficial]. Extracts a thin solid around every triangle: two walls around an open sheet, and an outer surface plus an inner one for a closed solid [S4, unofficial].
*   **Parameter:** `shell_voxels`: The shell half-thickness in voxels. Default value is $1.5f$. Must be $\ge \sqrt{3}/2$ ($0.866$) to prevent holes between voxels. Must be $<$ `trunc_dist` [S4, unofficial].
*   **History:** This is the unsigned offset field from Xu & Barbic, 'Signed Distance Fields for Polygon Soup Meshes' (GI 2014), before their interior-component removal step [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N^3 \cdot T)$, where $N$ is the grid resolution and $T$ is the number of triangles in the mesh [S2, S3, unofficial]. This is computationally expensive for high-resolution grids and complex meshes [S1].
*   **Memory Complexity:** $O(N^3)$ for storing the 3D texture [S3, unofficial].
*   **Performance:** For the Stanford Bunny mesh (35,947 vertices, 69,451 triangles) and an automobile part mesh (156,229 vertices, 312,458 triangles), the method requires significant computational resources [S1].
*   **Strengths:** Provides exact geometric distance to the mesh surface, allowing for high-quality rendering of implicit surfaces [S3, unofficial].
*   **Limitations:** Computationally intensive for large meshes and high resolutions. Accuracy depends on the quality of the scanned surface and the precision of the distance calculation [S1]. The need to close holes in the mesh adds a pre-processing step [S3, unofficial].

## Background
This method applies distance field sampling to scanned surfaces for isosurface extraction [S1]. The Stanford Bunny mesh, a standard test case with 35,947 vertices and 69,451 triangles, is frequently used to demonstrate this technique [S1][S3, unofficial]. The approach was implemented using tools like MeshDistance from the GEL (GEometry and Linear algebra) framework [S3, unofficial]. It is used in computer graphics for ray marching and implicit surface rendering [S2, S3, unofficial].

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S4] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
