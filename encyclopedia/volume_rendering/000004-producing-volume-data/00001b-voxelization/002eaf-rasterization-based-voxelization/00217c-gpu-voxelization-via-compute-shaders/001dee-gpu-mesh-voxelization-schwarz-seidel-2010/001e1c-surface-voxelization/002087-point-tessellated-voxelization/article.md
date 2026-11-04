# Point-Tessellated Voxelization

Also called: Hardware-based triangle subdivision and point-based surface voxelization

## Overview
Point-Tessellated Voxelization is a GPU-based framework for real-time surface voxelization that converts triangle meshes into a 3D texture grid by subdividing triangles via hardware tessellation, generating point primitives at their centroids, and rasterizing these points into volume texels [S1, unofficial]. It solves the problem of efficiently generating gap-free (watertight) voxel representations for dynamic or complex scenes where strict conservative accuracy is not required [S1, unofficial]. Unlike conservative voxelization methods that test triangle-voxel overlaps, this method relaxes the requirement to watertightness, ensuring no gaps by keeping subdivided triangles smaller than the target voxel cells, thereby avoiding expensive overlap tests [S1, unofficial]. It differs from naive pipeline-based voxelization, which only samples triangle centers and fails for triangles spanning multiple voxels, by using tessellation to uniformly sample points across the surface [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh positioned in world space and a target 3D voxel grid resolution with cell side length $l_{cell}$ [S1, unofficial].
*   **Output:** A 3D texture where each texel stores occupancy (binary) or multi-valued data (scalars/vectors like color or normal) [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D texture array or volume texture where each texel stores a value [S1, unofficial]. For binary voxelization, a constant non-zero value indicates occupancy [S1, unofficial]. For multi-valued voxelization, the texel stores scalars or vectors [S1, unofficial].
*   **Surfel:** A point primitive generated at the centroid of a subdivided triangle, representing a microtriangle [S1, unofficial].

**Parameters:**
*   $l_{cell}$: Side length of a voxel cell in world units [S1, unofficial].
*   $\gamma$: Global safety factor. $\gamma=1$ produces watertight results [S1, unofficial].
*   $A, B$: Empirical constants for bias estimation. $A = 1/p_1$ where $p_1$ is the bias proportion at $\gamma=1$. $B$ is estimated via expectation $E[\gamma/\ln(p/p_1)]$ [S1, unofficial].
*   Tessellation Limit: Hardware tessellation factors are typically limited (e.g., up to 63 in Direct3D 11), generating approximately 5000–6000 triangles per input triangle [S1, unofficial].

**Steps:**
1.  **Triangle Subdivision (Hull Shader & Tessellator):** Calculate tessellation factors for the three edges and the center of each input triangle to ensure subdivided patches are smaller than the voxel cell size [S1, unofficial]. Calculate edge factors $E_{ij} = \gamma |P_i - P_j| / l_{cell}$ for vertices $P_i, P_j$ [S1, unofficial]. Calculate edge tessellation factors $T_1 = \max_{x,y,z} E_{12}$, $T_2 = \max_{x,y,z} E_{23}$, $T_3 = \max_{x,y,z} E_{13}$ by taking the maximum projection along axes to determine occupied cells [S1, unofficial]. Calculate the central tessellation factor $T_C = (T_1 + T_2 + T_3)/3$ [S1, unofficial]. Pass $T_1, T_2, T_3, T_C$ to the fixed-function tessellator, which generates barycentric coordinates for tiled triangles [S1, unofficial].
2.  **Point Generation (Domain Shader & Geometry Shader):** The domain shader generates new vertices based on barycentric coordinates and control points [S1, unofficial]. The geometry shader receives the soup of subdivided triangles [S1, unofficial]. For each subdivided triangle with vertices $P_{vert1}, P_{vert2}, P_{vert3}$, calculate the centroid position $P_{centroid} = (P_{vert1} + P_{vert2} + P_{vert3})/3$ [S1, unofficial]. Replace the triangle with a single point primitive (surfel) at $P_{centroid}$ [S1, unofficial].
3.  **Point Injection/Voxelization (Geometry Shader & Pixel Shader):** In the geometry shader, transform $P_{centroid}$ from world space into the coordinate space of the 3D texture grid [S1, unofficial]. Calculate the integer texel indices $(i, j, k)$ corresponding to the surfel's position [S1, unofficial]. Output a value to the texel at $(i, j, k)$ in the pixel shader [S1, unofficial]. For binary voxelization, output a constant value to indicate occupancy [S1, unofficial]. For multi-valued voxelization, output scalars or vectors (e.g., color, normal) associated with the triangle [S1, unofficial]. For average values across multiple triangles, use additive blending in the pixel shader [S1, unofficial]. To preserve all point cloud values, compress them using spherical harmonics [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def create_3d_array(dims: tuple[int, int, int], init_val: float = 0.0):
    """Initialize a 3D array with dimensions (Nx, Ny, Nz) to init_val."""
    Nx, Ny, Nz = dims
    return [[[init_val for _ in range(Nz)] for _ in range(Ny)] for _ in range(Nx)]

def calculate_tessellation_factors(vertices: list[tuple[float, float, float]], l_cell: float, gamma: float):
    """
    vertices: List of 3 tuples [P1, P2, P3]
    l_cell: Side length of the voxel grid cell
    gamma: Safety factor (typically 1.0)
    Returns: (T1, T2, T3, TC)
    """
    P1, P2, P3 = vertices
    
    # Calculate edge lengths
    L12 = math.dist(P1, P2)
    L23 = math.dist(P2, P3)
    L13 = math.dist(P1, P3)
    
    # Calculate edge factors E_ij
    E12 = gamma * L12 / l_cell
    E23 = gamma * L23 / l_cell
    E13 = gamma * L13 / l_cell
    
    # Calculate edge tessellation factors T1, T2, T3
    # The source specifies max_{x,y,z} E_ij to determine occupied cells.
    # In a direct implementation, the scalar length ratio serves as the factor.
    T1 = E12
    T2 = E23
    T3 = E13
    
    # Calculate central factor
    TC = (T1 + T2 + T3) / 3.0
    
    return (T1, T2, T3, TC)

def generate_surfels(tessellated_triangles: list[list[tuple[float, float, float]]]):
    """
    tessellated_triangles: List of triangles, each with 3 vertices [V1, V2, V3]
    Returns: List of surfel positions (tuples)
    """
    surfels = []
    for tri in tessellated_triangles:
        V1, V2, V3 = tri
        # Calculate centroid
        centroid = tuple((V1[i] + V2[i] + V3[i]) / 3.0 for i in range(3))
        surfels.append(centroid)
    return surfels

def voxelize_surfels(surfels: list[tuple[float, float, float]], grid_origin: tuple[float, float, float], 
                     l_cell: float, grid_dims: tuple[int, int, int], mode: str):
    """
    surfels: List of 3D positions
    grid_origin: World space origin of the voxel grid
    l_cell: Side length of a voxel
    grid_dims: (Nx, Ny, Nz)
    mode: 'binary' or 'multi_valued'
    Returns: 3D array (Nx, Ny, Nz)
    """
    grid = create_3d_array(grid_dims, init_val=0.0)
    
    for p in surfels:
        # Transform to grid coordinates
        rel_pos = tuple(p[i] - grid_origin[i] for i in range(3))
        
        # Calculate texel indices
        ix = int(rel_pos[0] / l_cell)
        iy = int(rel_pos[1] / l_cell)
        iz = int(rel_pos[2] / l_cell)
        
        # Clamp to grid bounds to avoid out-of-bounds access
        if 0 <= ix < grid_dims[0] and 0 <= iy < grid_dims[1] and 0 <= iz < grid_dims[2]:
            if mode == 'binary':
                # Mark as occupied
                grid[ix][iy][iz] = 1.0
            elif mode == 'multi_valued':
                # Additive blending for average values
                grid[ix][iy][iz] += 1.0
                
    return grid
```

**Edge Cases and Pitfalls:**
*   **Bias:** The method is not conservative and may miss voxels [S1, unofficial].
*   **Small Triangles:** A triangle smaller than a voxel may only detect one voxel via its center, missing others [S1, unofficial].
*   **Subdivision Level Loss:** Voxels detected at a lower subdivision level may be lost at a higher subdivision level [S1, unofficial].
*   **Performance Degradation:** Hardware tessellation performance degrades significantly when increasing grid size from $128^3$ to $1024^3$ on older hardware due to high tessellation factors [S1, unofficial].
*   **Low Triangle Count:** Performance advantage diminishes when the number of triangles is very low while grid resolution is high [S1, unofficial].
*   **Displacement Artifacts:** When using displacement mapping, the original displacement map may not be perpendicular to new subdivided triangles, introducing artifacts, though experiments showed minimal impact [S1, unofficial].

## Subtopics

### [Parametric Surface Voxelization](2389-parametric-surface-voxelization.md)
Parametric surface voxelization is a GPU-based technique that converts parametric surface patches directly into a voxel grid by rasterizing point primitives generated in the parametric domain. It differs from standard triangle rasterization by operating in the 2D parameter space of the surface definition, allowing for the voxelization of complex implicit or free-form surfaces that lack explicit triangular meshes. This method enables real-time voxelization of complex models by leveraging the vertex shader to map 2D texture coordinates to 3D world positions.

## Operating characteristics
*   **Time Complexity:** The method is efficient, requiring only one rendering pass, with performance scaling with the number of triangles and the tessellation factor [S1, unofficial]. On a GTX480, processing 22.1k triangles takes 0.46 ms at $128^3$ resolution and 9.10 ms at $512^3$ resolution [S1, unofficial].
*   **Memory Complexity:** Limited only by video memory, with the voxel grid size determined by the resolution and multiple values stored per voxel [S1, unofficial].
*   **Accuracy vs. Performance:** Using both vertices and face centroids improves accuracy by approximately 10% but causes approximately 40% performance loss, so the centroid alone is chosen as the empirical balance [S1, unofficial].
*   **Strengths:** Real-time performance, gap-free results with $\gamma=1$, support for multi-valued data, handling large grids limited by video memory, and convenience compared to slice-based or bitwise methods [S1, unofficial].
*   **Limitations:** Not conservative and may miss voxels, tessellation performance degrades on high-resolution grids on older hardware, not suitable for applications requiring perfect accuracy, and bias increases with larger grid sizes or coarse meshes with displacement [S1, unofficial].
*   **When to Use:** Real-time applications like video games, virtual reality, and hybrid ambient occlusion where visual quality is prioritized over strict geometric accuracy [S1, unofficial].
*   **When Not to Use:** Applications requiring conservative voxelization such as precise collision detection or solid modeling [S1, unofficial].

## Background
Point-Tessellated Voxelization integrates hardware-based triangle subdivision (Direct3D 11/OpenGL 4.0) with point-based rendering to achieve real-time surface voxelization [S1, unofficial]. The method was designed to address the need for interactive voxel generation for applications such as shape matching, visibility processing, rapid manufacturing, and 360-degree display [S1, unofficial]. It builds on the concept of point-based rendering and leverages the GPU's tessellation pipeline to sample surfaces uniformly [S1, unofficial]. The technique has been applied in hybrid ambient occlusion (HAO), substituting the voxelization process for ray marching [S1, unofficial].

## Sources
[S1] Point-Tessellated Voxelization, http://www.cs.columbia.edu/cg/raymond/gi12/paper100.pdf, fetched 2026-10-26 (unofficial)
