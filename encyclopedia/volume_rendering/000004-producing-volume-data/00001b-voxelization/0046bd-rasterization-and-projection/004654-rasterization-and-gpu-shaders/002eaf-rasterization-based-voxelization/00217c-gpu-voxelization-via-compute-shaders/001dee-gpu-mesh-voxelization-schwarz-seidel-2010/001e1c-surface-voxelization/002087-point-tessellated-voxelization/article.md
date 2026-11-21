# Point-Tessellated Voxelization

Also called: Hardware-based triangle subdivision and point-based surface voxelization

## Overview
Point-Tessellated Voxelization is a GPU-based framework for real-time multi-valued surface voxelization that uses hardware tessellation to subdivide triangles, generates point primitives at the centroids of subdivided triangles, and rasterizes these points into a volume texture to achieve gap-free (watertight) voxelization [S1, unofficial]. It solves the problem of efficiently converting triangle meshes into voxel grids for applications like hybrid ambient occlusion, where strict conservative accuracy is not required but performance is critical [S1, unofficial]. It sets itself apart from naive pipeline-based voxelization by sampling multiple points per triangle via tessellation to handle triangles occupying multiple voxels, and from conservative methods by relaxing the requirement to watertight (gap-free) rather than covering all overlapping voxels, thus avoiding expensive triangle-voxel overlap tests [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh (vertices $P_i$) and a 3D texture grid of resolution $N^3$ with cell side length $l_{cell}$ [S1, unofficial].
*   **Output:** A 3D texture where each texel stores occupancy (binary) or scalar/vector data (multi-valued) [S1, unofficial].

**Data Structures:**
*   **3D Texture Grid:** Stores occupancy or compressed scalar/vector data per texel [S1, unofficial]. The grid size is limited only by video memory [S1, unofficial].
*   **Surfel:** A point primitive placed at the centroid of a subdivided triangle [S1, unofficial].

**Parameters:**
*   **Global Safety Factor ($\gamma$):** Controls the balance between performance and accuracy. $\gamma=1$ produces watertight results [S1, unofficial].
*   **Bias Proportion ($p$):** The ratio of missing voxels to occupied voxels in the ground truth [S1, unofficial].
*   **Empirical Constants ($A, B$):** Used to compute $\gamma$ from a desired bias $p$. $A = 1/p_1$ where $p_1$ is the bias at $\gamma=1$. $B = -E(\gamma/\ln(p/p_1))$ [S1, unofficial].
*   **Grid Resolution:** Tested at $128^3, 512^3, 1024^3$ [S1, unofficial].

**Numbered Steps:**

1.  **Triangle Subdivision (Hull Shader & Tessellator):**
    Calculate tessellation factors for the input mesh using hardware tessellation support (Direct3D 11 or OpenGL 4.0) [S1, unofficial].
    *   Calculate edge factors $E_{ij}$ for each edge $(i,j)$ of the triangle:
        $$ E_{ij} = \frac{\gamma |P_i - P_j|}{l_{cell}} $$
        where $P_i, P_j$ are vertex positions and $l_{cell}$ is the voxel grid cell side length [S1, unofficial].
    *   Compute the maximum projection along axes for each edge to determine occupied cells:
        $$ T_1 = \max_{x,y,z} E_{12}, \quad T_2 = \max_{x,y,z} E_{23}, \quad T_3 = \max_{x,y,z} E_{13} $$
    *   Compute the central tessellation factor:
        $$ T_C = (T_1 + T_2 + T_3)/3 $$
    *   The fixed-function tessellator subdivides the triangle based on $T_C$ [S1, unofficial].

2.  **Point Generation (Domain Shader & Geometry Shader):**
    *   The domain shader generates new vertices based on barycentric coordinates and control points [S1, unofficial].
    *   The geometry shader receives the tessellated triangle soup [S1, unofficial].
    *   Replace each subdivided triangle with a single point primitive (surfel) placed at its centroid [S1, unofficial].
    *   Calculate the centroid position in world space:
        $$ P_{centroid} = (P_{vert1} + P_{vert2} + P_{vert3})/3 $$
    *   This point is empirically chosen as the balance between performance and accuracy compared to vertices or incenter [S1, unofficial].

3.  **Point Injection/Voxelization (Geometry Shader & Pixel Shader):**
    *   Transform the point cloud into the coordinate space of the 3D texture grid within the geometry shader [S1, unofficial].
    *   For each surfel, calculate the position of the corresponding texel in the 3D texture [S1, unofficial].
    *   Output a value into the calculated texel in the pixel shader [S1, unofficial].
    *   **Binary Voxelization:** Output a constant value to indicate occupancy [S1, unofficial].
    *   **Multi-valued Voxelization:** Store scalars or vectors (colors, normals) from triangles into voxels [S1, unofficial].
        *   For average values across multiple triangles, use additive blending [S1, unofficial].
        *   To preserve all point cloud values, compress them using spherical harmonics [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_tessellation_factors(vertices, gamma, l_cell):
    # vertices: list of 3 vectors [P1, P2, P3]
    # gamma: safety factor (default 1.0)
    # l_cell: side length of a voxel cell
    
    # Calculate edge lengths
    e12 = length(vertices[1] - vertices[0])
    e23 = length(vertices[2] - vertices[1])
    e13 = length(vertices[2] - vertices[0])
    
    # Calculate edge factors
    E12 = gamma * e12 / l_cell
    E23 = gamma * e23 / l_cell
    E13 = gamma * e13 / l_cell
    
    # Compute central factor
    # The source defines T1, T2, T3 as max over axes. 
    # If E_ij is a scalar, max over axes is just E_ij.
    # If E_ij is a vector (projection), we take max component.
    # Given the formula E_ij = gamma * |Pi - Pj| / l_cell, it is a scalar.
    # Thus T1 = E12, T2 = E23, T3 = E13.
    
    T1 = E12
    T2 = E23
    T3 = E13
    
    T_C = (T1 + T2 + T3) / 3
    
    return T_C

def generate_surfels(tessellated_triangles):
    surfels = []
    for tri in tessellated_triangles:
        # tri is a list of 3 vertices [v1, v2, v3]
        centroid = (tri[0] + tri[1] + tri[2]) / 3.0
        surfels.append(centroid)
    return surfels

def voxelize_surfels(surfels, grid_resolution, grid_origin, grid_scale):
    # grid_resolution: N (N^3 grid)
    # grid_origin: world space origin of the grid
    # grid_scale: world space size of the grid (or l_cell)
    
    # Initialize 3D texture (volume)
    volume = [[[0.0 for _ in range(grid_resolution)] for _ in range(grid_resolution)] for _ in range(grid_resolution)]
    
    for p in surfels:
        # Transform point to grid coordinates
        # Assuming grid_origin is the min corner and grid_scale is the max corner
        # Or grid_scale is l_cell. Let's assume grid_scale is l_cell for direct indexing.
        
        # Calculate texel index
        # p is in world space.
        # idx = floor((p - grid_origin) / l_cell)
        
        # Using l_cell from the tessellation step for consistency
        # Note: grid_origin and l_cell must be consistent with the tessellation context
        
        idx_x = floor((p.x - grid_origin.x) / grid_scale)
        idx_y = floor((p.y - grid_origin.y) / grid_scale)
        idx_z = floor((p.z - grid_origin.z) / grid_scale)
        
        # Check bounds
        if 0 <= idx_x < grid_resolution and \
           0 <= idx_y < grid_resolution and \
           0 <= idx_z < grid_resolution:
            
            # Binary voxelization: mark as occupied
            volume[idx_x][idx_y][idx_z] = 1.0
            
            # Multi-valued voxelization would store additional data here
            # e.g., using additive blending or spherical harmonics compression
            
    return volume
```

**Edge Cases and Pitfalls:**
*   **Accuracy vs. Performance:** Using both vertices and face centroids improves accuracy by ~10% but causes ~40% performance loss. The centroid alone is chosen as the empirical balance [S1, unofficial].
*   **Not Conservative:** The method is watertight but not conservative; it may miss some voxels. Bias ranges from 1.7% to 4.4% across tested scenes [S1, unofficial].
*   **Small Triangles:** A triangle smaller than a voxel may only detect one voxel via its center, missing others [S1, unofficial].
*   **Subdivision Level Loss:** Voxels detected at a lower subdivision level may be lost at a higher subdivision level [S1, unofficial].
*   **Hardware Limits:** Hardware tessellation performance degrades significantly when increasing grid size from $128^3$ to $1024^3$ on GTX480 due to high tessellation factors [S1, unofficial].
*   **Bias Increase:** Bias increases with larger grid sizes or when applying displacement voxelization to coarse meshes. Bias also increases when the number of triangles is very low while the grid resolution is high [S1, unofficial].
*   **Multi-Pass Subdivision:** If the mesh has very few triangles (e.g., a tetrahedron) and the grid is large, multiple subdivision passes may be needed because hardware tessellation factors are limited (up to 63) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is efficient, requiring only one rendering pass [S1, unofficial]. Performance scales with the number of triangles and grid resolution [S1, unofficial].
*   **Memory Complexity:** Limited only by video memory for the 3D texture grid [S1, unofficial].
*   **Measured Performance (GTX480, Core-i7 920):**
    *   871k tris: 0.4 ms ($128^3$), 2.3 ms ($512^3$), 12.8 ms ($1024^3$) [S1, unofficial].
    *   7.2M tris: 0.4 ms ($128^3$), 2.7 ms ($512^3$), 9.7 ms ($1024^3$) [S1, unofficial].
    *   2.88M tris: 5.9 ms ($128^3$), 7.5 ms ($512^3$), 56.8 ms ($1024^3$) [S1, unofficial].
    *   1.76M tris: 0.5 ms ($128^3$), 5.3 ms ($512^3$), 7.3 ms ($1024^3$) [S1, unofficial].
    *   General framework cost: 0.46 ms ($128^3$), 9.10 ms ($512^3$) for 22.1k triangles [S1, unofficial].
*   **Scaling:** Tessellation cost increases sharply with grid size on older hardware (GTX480) [S1, unofficial].
*   **Strengths:**
    *   Real-time performance [S1, unofficial].
    *   Simple shader programming implementation [S1, unofficial].
    *   Supports multi-valued voxelization [S1, unofficial].
    *   Handles arbitrary deformation (recomputable per frame) [S1, unofficial].
    *   Faster than directly voxelizing high-detail meshes (4x speedup with displacement mapping) [S1, unofficial].
*   **Limitations:**
    *   Not conservative (may miss voxels) [S1, unofficial].
    *   Accuracy degrades with very low triangle counts and high grid resolutions [S1, unofficial].
    *   Tessellation performance bottleneck on older hardware at high resolutions [S1, unofficial].
    *   Not suitable for applications requiring perfect accuracy (e.g., collision detection) [S1, unofficial].
*   **When to Use:** Real-time applications like video games, VR, and Hybrid Ambient Occlusion (HAO) where visual quality is acceptable with minor artifacts [S1, unofficial].
*   **When Not to Use:** Applications requiring conservative voxelization (e.g., precise collision detection) or volumetric/solid voxelization [S1, unofficial].

## Background
*   **Origin:** Proposed by Yun et al. [S1, unofficial].
*   **History:** The method integrates hardware-based triangle subdivision (Direct3D 11/OpenGL 4.0) and point-based rendering into a framework for surface voxelization. It was designed to address the need for real-time voxelization in applications like shape matching, visibility processing, and rapid manufacturing [S1, unofficial].
*   **Usage:** Used for Hybrid Ambient Occlusion (HAO) by substituting the voxelization process for ray marching [S1, unofficial].

## Sources
[S1] Point-Tessellated Voxelization, http://www.cs.columbia.edu/cg/raymond/gi12/paper100.pdf, fetched 2026-10-26 (unofficial)
