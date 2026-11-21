# GPU-based SDF Generation

Also called: Compute shader SDF generation, GPU-based Signed Distance Field Generation, SDF generation on GPU

## Overview
GPU-based SDF Generation is a technique to compute Signed Distance Fields (SDFs) for a mesh by uploading triangle vertex data to a texture and calculating distances in a pixel or compute shader, offering significant speedup over CPU-based methods [S1, unofficial]. It solves the performance bottleneck of CPU voxelization by leveraging massive parallelism, enabling real-time or near-real-time SDF updates for animated characters [S1, unofficial]. Unlike [Naive SDF Generation (CPU)](10688-naive-sdf-generation-cpu.md), which processes voxels sequentially or with limited threading, this method maps the distance calculation workload directly to the GPU's parallel cores, reducing generation time from seconds to milliseconds [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A mesh defined by $N$ triangles, where each triangle has 3 vertices. Each vertex has a 3D position $(x, y, z)$.
- Output: A 3D texture (volume) of resolution $R_x \times R_y \times R_z$, where each voxel stores the signed distance to the mesh surface.

**Data Structures:**
1.  **Triangle Texture:** A 2D texture storing vertex positions.
    -   Size: The texture width $W$ and height $H$ must accommodate $3N$ vertices.
    -   Layout: Vertices are stored sequentially. For triangle $i$, vertices $3i, 3i+1, 3i+2$ are stored in consecutive texture elements.
    -   Format: Typically `RGBA32F` or `RGB32F` to store 3D coordinates.
    -   Constraint: The texture width $W$ is limited by the GPU's maximum texture dimension (e.g., 16384 for D3D11) [S1, unofficial]. If $3N > W \times H_{max}$, the data must be wrapped across multiple textures or a larger height is used, limited by $W \times H \ge 3N$ [S1, unofficial].
2.  **Output Render Target:** A 3D texture or a series of 2D slices representing the SDF volume.

**Numbered Steps:**
1.  **Prepare Triangle Data:** Collect all vertex positions from the mesh.
2.  **Upload to Texture:** Create a 2D texture with sufficient capacity to hold $3N$ vertices. Upload the vertex positions into this texture.
    -   *Note:* Precomputing edge vectors or normals is not recommended because GPUs take longer to read texture data than to compute simple formulas [S1, unofficial].
3.  **Setup Render Target:** Create a 3D texture (or 2D array) for the SDF output with the desired voxel resolution.
4.  **Dispatch Shader:** Execute a pixel shader or compute shader for each voxel in the output volume.
5.  **Calculate Distance:** For each voxel at position $\mathbf{p}$:
    -   Iterate through all triangles in the triangle texture.
    -   For each triangle with vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$, calculate the distance from $\mathbf{p}$ to the triangle.
    -   Keep track of the minimum distance found so far.
6.  **Determine Sign:** Determine the sign of the distance (inside/outside).
    -   *Initial Approach:* The author initially attempted to determine the sign by comparing the distance to neighbors, but this failed because faulty areas could cover more than one pixel's width [S1, unofficial].
    -   *Robust Approach:* Cast a ray from the sample point $\mathbf{p}$ in a fixed direction (e.g., upwards) and count intersections with all triangles. If the number of intersections is even, the point is outside (positive distance); if odd, it is inside (negative distance) [S1, unofficial]. The Müller-Trombose algorithm is suggested for intersection testing [S1, unofficial].
7.  **Store Result:** Write the signed distance to the output volume.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the GPU shader (Compute or Pixel Shader)
# Input: TriangleTexture (stores 3 vertices per triangle), OutputVolume (SDF)
# Parameters: NumTriangles, VoxelResolution

def ComputeSDF(VoxelPosition p):
    min_dist = infinity
    num_intersections = 0
    
    # Iterate over all triangles
    for i from 0 to NumTriangles - 1:
        # Fetch vertex positions from texture
        v0 = TriangleTexture.Sample(i * 3).xyz
        v1 = TriangleTexture.Sample(i * 3 + 1).xyz
        v2 = TriangleTexture.Sample(i * 3 + 2).xyz
        
        # Calculate distance from p to triangle v0-v1-v2
        dist = DistancePointToTriangle(p, v0, v1, v2)
        
        # Update minimum distance
        if dist < min_dist:
            min_dist = dist
            
        # Count intersections for sign determination
        # Cast ray from p in a fixed direction (e.g., +Y)
        if RayIntersectsTriangle(p, direction=(0,1,0), v0, v1, v2):
            num_intersections += 1
            
    # Determine sign
    if num_intersections % 2 == 0:
        sign = 1.0
    else:
        sign = -1.0
        
    return sign * min_dist

def DistancePointToTriangle(p, v0, v1, v2):
    # Standard point-to-triangle distance calculation
    # Returns the Euclidean distance from p to the closest point on the triangle
    # This involves projecting p onto the plane, checking barycentric coordinates,
    # and handling edge/vertex cases.
    # Implementation details omitted for brevity, but standard geometric formula.
    pass

def RayIntersectsTriangle(ray_origin, ray_direction, v0, v1, v2):
    # Müller-Trombose or Möller-Trumbore algorithm
    # Returns true if the ray intersects the triangle
    pass
```

**Parameters:**
-   **Max Triangles:** Limited by the texture size. For a maximum texture width of 16384 (D3D11 limit), the maximum number of triangles is approximately $16384 \times 16384 / 3 \approx 89.5$ million [S1, unofficial].
-   **Texture Width Limit:** 16384 pixels (D3D11) [S1, unofficial]. If the required width exceeds this, the data must be wrapped to a new line (increasing height) [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Sign Flipping Artifacts:** Pixels may have incorrect signs if the closest vertex is shared by triangles with different normal directions, leading to an incorrect sign choice based on local proximity [S1, unofficial].
-   **Neighbor Comparison Failure:** Comparing SDF values to neighbors to fix signs is unreliable because faulty areas can span multiple pixels [S1, unofficial].
-   **Performance Bottleneck:** The limiting factor may be copying the texture and setting up render targets rather than the computation itself, especially for smaller resolutions [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of triangles and $V$ is the number of voxels. Each voxel must check all triangles.
-   **Memory Complexity:** $O(N + V)$. $O(N)$ for the triangle texture, $O(V)$ for the output SDF volume.
-   **Measured Performance:**
    -   For a 50x50x50 SDF of a tiger model: GPU generation took 2ms, while CPU generation with 4 threads took 60610ms [S1, unofficial].
    -   For a 100x100x100 SDF: GPU generation took 5ms [S1, unofficial].
-   **Strengths:** Extremely fast generation times compared to CPU methods, enabling real-time updates for animated characters [S1, unofficial].
-   **Limitations:** Limited by GPU texture size constraints (max ~89.5 million triangles for D3D11) [S1, unofficial]. Sign determination can be complex and may require ray casting, adding computational overhead. Artifacts may occur if sign determination is not robust [S1, unofficial].
-   **When to Use:** When real-time or near-real-time SDF generation is required, or when the mesh is too large for efficient CPU processing.
-   **When Not to Use:** When the mesh exceeds GPU texture size limits, or when the overhead of texture setup and copying outweighs the computation time (e.g., very small resolutions) [S1, unofficial].

## Background
This method was described by kosmonaut in a blog post titled "Signed Distance Field Rendering Journey pt.1" [S1, unofficial]. The author initially attempted CPU-based SDF generation but found it too slow. After realizing that computing the texture on the GPU is trivial, they implemented this method, achieving significant speedups [S1, unofficial]. The approach was developed as part of a journey to render SDFs, with the goal of enabling real-time updates for animated characters [S1, unofficial].

## Sources
[S1] Signed Distance Field Rendering Journey pt.1 – kosmonaut's blog, https://kosmonautblog.wordpress.com/2017/05/01/signed-distance-field-rendering-journey-pt-1/, fetched 2026-10-29 (unofficial)
