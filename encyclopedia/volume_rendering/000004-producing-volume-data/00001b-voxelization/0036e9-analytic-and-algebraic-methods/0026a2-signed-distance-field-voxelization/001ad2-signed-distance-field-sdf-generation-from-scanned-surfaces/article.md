# Signed Distance Field (SDF) Generation from Scanned Surfaces

Also called: 3D Texture Distance Field for Implicit Surfaces, Mesh SDF Generation, Mesh to SDF Voxelization, MeshDistance conversion, SDF Texture Generation from Mesh, SDF Voxelization, Signed distance to triangulated surface

## Overview
This method converts a triangulated surface mesh (typically from 3D scanning) into a 3D Signed Distance Field (SDF) stored as a texture or grid. It solves the problem of representing explicit, potentially noisy or complex scanned geometry as an implicit function suitable for raymarching, isosurfacing, or volume rendering. Unlike [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md), which processes depth slices, this method operates directly on geometric primitives (triangles) to compute exact Euclidean distances [S1]. It differs from [Signed Distance Field from Mesh](1327-signed-distance-field-from-mesh.md) by explicitly handling the conversion to a bounded 3D texture domain with specific boundary shell logic [S3, unofficial].

## How it works
**Inputs:**
*   A triangulated mesh (e.g., PLY format) with vertices and triangles [S1].
*   Grid resolution $N$ (e.g., $N=128$ for tetrahedral grids to preserve detail) [S1].
*   Texture boundary parameters: `textureBoundary` (default $0.5$) and `textureShell` (default $0.05$) [S3, unofficial].

**Outputs:**
*   A 3D texture/grid where each voxel stores the signed distance to the mesh surface. In GPU implementations, this is often stored in the red channel of a fragment shader output [S2, unofficial].

**Data Structures:**
*   **Mesh:** Stored as a 1D array of vertices with a stride (e.g., 16384 bytes per vertex in some GPU implementations) [S2, unofficial].
*   **Grid:** A 3D array indexed by integer coordinates `pos` derived from normalized texture coordinates. The domain is typically normalized to $[-1, 1]$ or mapped from $[0, 1]$ [S2, unofficial][S3, unofficial].

**Steps:**

1.  **Pre-processing:** Close any holes in the input mesh (e.g., the 5 holes in the Stanford Bunny) to ensure correct topological containment and distance representation [S3, unofficial].
2.  **Grid Initialization:** Define the bounding box centered at the origin, bounded by $|x| \le \text{textureBoundary}$ (typically a $1 \times 1 \times 1$ cube) [S3, unofficial].
3.  **Distance Computation:** For each voxel center $x$ in the grid:
    *   Iterate through all triangles in the mesh.
    *   Compute the Euclidean distance $d$ from $x$ to the triangle surface using a routine like `dist3d` or `udTriangle` [S1][S2, unofficial].
    *   Update the minimum distance: $D(x) = \min(D(x), d)$ [S2, unofficial].
    *   Determine the sign: Negative if inside the surface, positive if outside [S1]. This is often achieved via ray-casting parity or winding number, though the specific sign algorithm is delegated to the `dist3d` routine [S1].
4.  **Boundary Handling (Texture Lookup):** When evaluating the distance field at runtime or for boundary points, use the following logic to handle points outside the valid texture region [S3, unofficial]:
    *   Calculate difference: $d_i = |x| - \text{textureBoundary} + \text{textureShell}$.
    *   If the point is outside the 3D texture space (based on $d_i$), calculate the distance as the L2 norm of the positive components: $res = \| \max(d_i + \text{textureShell}, 0) \|_2$.
    *   If the point is inside the texture space (check if $c = \max(d_{i_x}, \max(d_{i_y}, d_{i_z})) < 0$):
        *   Adjust coordinate: $x' = x + \text{textureBoundary}$.
        *   Lookup the pre-computed distance from the 3D texture: $res = \text{lookupTexture}(\text{texture}, x')$.
    *   Return $res$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computation phase (CPU or GPU Compute Shader)
def generate_sdf_from_mesh(mesh, resolution, texture_boundary=0.5, texture_shell=0.05):
    """
    Generates a 3D SDF texture from a triangulated mesh.
    mesh: List of triangles, each defined by 3 vertices (vec3).
    resolution: Integer N, defining an N x N x N grid.
    """
    # Initialize grid with a large value
    grid_size = resolution * resolution * resolution
    sdf_grid = [10000.0] * grid_size
    
    # Normalize grid coordinates to [-1, 1] range for distance calculation
    # The domain is a cube centered at origin with side length 2*texture_boundary
    step = (2.0 * texture_boundary) / resolution
    
    for z in range(resolution):
        for y in range(resolution):
            for x in range(resolution):
                # Map grid index to world coordinate
                # Assuming grid indices 0..N-1 map to -boundary..+boundary
                coord_x = (x / resolution) * 2.0 - 1.0
                coord_y = (y / resolution) * 2.0 - 1.0
                coord_z = (z / resolution) * 2.0 - 1.0
                
                # Scale to actual boundary size
                pos = vec3(coord_x * texture_boundary, 
                           coord_y * texture_boundary, 
                           coord_z * texture_boundary)
                
                # Compute minimum distance to mesh surface
                min_dist = 10000.0
                is_inside = False
                
                # Iterate over all triangles
                for triangle in mesh.triangles:
                    # Compute distance from point to triangle
                    d = distance_point_to_triangle(pos, triangle.v0, triangle.v1, triangle.v2)
                    if d < min_dist:
                        min_dist = d
                        # Determine sign based on ray casting or normal projection
                        # [G] Specific sign determination algorithm (e.g., parity test) 
                        # is implementation-dependent but required for signed distance.
                        # Here we assume a helper function returns signed distance.
                        signed_d = compute_signed_distance(pos, triangle)
                        if signed_d < min_dist:
                            min_dist = signed_d
                
                # Store in grid
                grid_index = x + y * resolution + z * resolution * resolution
                sdf_grid[grid_index] = min_dist
                
    return sdf_grid

# Runtime evaluation phase (Shader)
def evaluate_sdf_at_point(x, sdf_texture, texture_boundary=0.5, texture_shell=0.05):
    """
    Evaluates the SDF at a point x, handling boundary conditions.
    """
    # Calculate distance to the texture boundary box
    di = abs(x) - texture_boundary + texture_shell
    
    # Check if point is outside the 3D texture space
    # If any component of di + texture_shell is positive, it's outside
    if any(di + texture_shell > 0):
        # Distance is the L2 norm of the positive components
        res = length(max(di + texture_shell, 0.0))
    else:
        # Point is inside the texture space
        # Adjust coordinate to map to texture lookup range [0, 1] or similar
        # Assuming texture is stored in [-boundary, boundary] mapped to [0,1]
        x_adjusted = x + texture_boundary
        # Normalize to [0, 1] for texture lookup if needed
        # [G] Exact normalization depends on texture coordinate system.
        # Assuming texture covers [-boundary, boundary]
        tex_coord = (x_adjusted / (2.0 * texture_boundary)) + 0.5
        
        # Lookup pre-computed distance
        res = texture(sdf_texture, tex_coord).r
        
    return res
```

**Parameters:**
*   `textureBoundary`: Defines the box centered at origin bounded by $|x| \le 0.5$. Represents the full $1 \times 1 \times 1$ cube containing the texture [S3, unofficial].
*   `textureShell`: A small value (default $0.05$) defining a boundary region around the texture where distance values might be invalid. It should be as small as possible to maximize the valid lookup region [S3, unofficial].
*   `shell_voxels`: (Variant specific) The shell half-thickness in voxels. Default $1.5$. Must be $\ge \sqrt{3}/2 \approx 0.866$ to prevent holes [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes:** Open meshes (e.g., Stanford Bunny with 5 holes) must be closed before conversion to ensure correct interior/exterior sign assignment [S3, unofficial].
*   **Boundary Accuracy:** The `textureShell` value is not universal and must be adjusted for different textures to balance avoiding invalid boundary data and maximizing the valid lookup area [S3, unofficial].
*   **Scanner Noise:** The accuracy of the resulting SDF depends heavily on the surface scanner quality and the accuracy of the signed distance computation routine [S1].
*   **Resolution:** Low resolution may lose detail. For complex meshes, resolution may need to be increased (e.g., $N=128$) [S1].

## Subtopics

### Shell SDF
An unsigned distance field minus a shell half-thickness. Negative within `shell_voxels` of any triangle, positive beyond. Works on open, non-manifold, self-intersecting, and inconsistently wound meshes. Extracts a thin solid around every triangle: two walls around an open sheet, and an outer surface plus an inner one for a closed solid [S4, unofficial]. Default `shell_voxels` is $1.5$, must be $\ge \sqrt{3}/2$ and $< \text{trunc\_dist}$ [S4, unofficial]. This is the unsigned offset field from Xu & Barbic, 'Signed Distance Fields for Polygon Soup Meshes' (GI 2014), before their interior-component removal step [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N^3 \cdot T)$, where $N$ is the grid resolution and $T$ is the number of triangles. This is computationally expensive for high-resolution grids and complex meshes [S1][S2, unofficial].
*   **Memory Complexity:** $O(N^3)$ to store the 3D texture/grid [S3, unofficial].
*   **Performance:** Measured on standard meshes like the Stanford Bunny (35,947 vertices, 69,451 triangles) and automobile parts (156,229 vertices, 312,458 triangles) [S1]. Performance scales linearly with triangle count and cubicly with resolution.
*   **Strengths:** Provides exact Euclidean distances to the mesh surface, enabling high-quality rendering and isosurfacing. Handles complex geometry directly [S3, unofficial].
*   **Limitations:** Computationally intensive for large meshes. Sensitive to mesh quality (holes, normals). Boundary handling requires careful tuning of `textureShell` [S3, unofficial].
*   **When to use:** When exact distance to a scanned mesh is required for raymarching or implicit surface reconstruction.
*   **When not to use:** For real-time applications with very high triangle counts where approximate methods (like voxelization) are preferred, or when the mesh is not closed and topological containment is ambiguous.

## Background
The method originates from the need to represent complex scanned surfaces (like the Stanford Bunny) as implicit distance functions for rendering and simulation. Early implementations used tools like `MeshDistance` from the GEL (GEometry and Linear algebra) framework to sample distances into 3D textures [S3, unofficial]. The approach was popularized in the context of GPU raymarching, where distance fields allow for efficient scene representation [S2, unofficial]. The specific boundary handling algorithm (Algorithm 4.1) was detailed in academic work on GPU ray marching of distance fields [S3, unofficial]. The Shell SDF variant was introduced by Xu & Barbic in 'Signed Distance Fields for Polygon Soup Meshes' (GI 2014) to handle non-manifold meshes [S4, unofficial].

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S4] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
