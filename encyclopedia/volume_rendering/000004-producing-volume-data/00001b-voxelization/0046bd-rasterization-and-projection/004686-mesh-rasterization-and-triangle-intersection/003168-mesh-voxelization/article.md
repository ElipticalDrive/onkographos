# Mesh Voxelization

Also called: STL Mesh Voxelization, Raycasting-based Voxelization, VoxelGrid from Mesh (Ray Method)

## Overview
Mesh Voxelization converts a continuous triangular surface mesh (e.g., STL, OBJ) into a discrete 3D voxel grid by determining the occupancy of each grid cell via ray-mesh intersection tests [S1, unofficial][S4, unofficial]. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which projects geometry onto a 2D plane and extrudes coverage, this method evaluates the topological relationship (inside/outside) of the voxel center relative to the closed surface using the ray casting parity rule [S1, unofficial][S2, unofficial]. It serves as the foundational technique for generating solid or surface-only voxel representations from polygonal models, distinct from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) which operates on discrete samples rather than continuous geometry [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangular mesh defined by vertex positions and triangle indices; a grid resolution $N$ (or voxel size $s$); a bounding box $[B_{min}, B_{max}]$; and optional parameters for sampling and interior filling [S3, unofficial][S4, unofficial].
*   **Output:** A voxel grid where each cell $(i, j, k)$ contains a boolean occupancy flag, and optionally a Signed Distance Field (SDF) value or attribute data [S4, unofficial].

**Data Structures:**
1.  **Mesh Data:** A flat array of vertices (stored as doubles in some implementations) and indices (3 integers per triangle) [S4, unofficial].
2.  **Voxel Grid:** A 3D array of size $N_x \times N_y \times N_z$ [S4, unofficial]. In binary formats (e.g., binvox), this is often packed into bits or bytes [S2, unofficial].
3.  **Ray Structure:** Origin $O$ (voxel center) and Direction $D$ (unit vector) [S2, unofficial].

**Coordinate Mapping:**
The grid origin is typically aligned with the mesh bounding box [S2, unofficial]. For a resolution $N$ and bounding box extent $E = \max(B_{max} - B_{min})$, the voxel size is $s = E / (N - 2)$ (or similar scaling depending on implementation) [S2, unofficial]. The center of voxel $(i, j, k)$ is computed as:
$$ C_{ijk} = B_{min} + (i + 0.5) \cdot s, \quad i \in [0, N_x-1] $$
(Analogous for $j, k$) [S4, unofficial].

**Algorithm Steps:**

1.  **Preprocessing:**
    *   Load mesh vertices and triangles. Validate that index arrays are multiples of 3 [S4, unofficial].
    *   Compute the Axis-Aligned Bounding Box (AABB) of the mesh [S4, unofficial].
    *   Define grid dimensions $N_x, N_y, N_z$ based on resolution and bounds [S4, unofficial].
    *   If interior filling is required, expand bounds by a `narrow_band` distance if specified [S4, unofficial].

2.  **Surface Occupancy (Ray Casting):**
    For each voxel center $C_{ijk}$:
    *   Cast a ray $R(t) = C_{ijk} + t \cdot D$ [S4, unofficial].
    *   Intersect the ray with all mesh triangles using the Möller–Trumbore algorithm [S2, unofficial].
    *   Collect all intersection distances $t$ [S4, unofficial]. Sort them to handle precision errors [S4, unofficial].
    *   Count the number of intersections $N_{hits}$ [S4, unofficial].
    *   Determine parity: If $N_{hits}$ is odd, the voxel is inside; if even, it is outside [S4, unofficial].
    *   **Double-thick handling:** For meshes with double-sided geometry (e.g., thin shells), check if $(N_{hits} / 2) \% 2 == 1$ [S2, unofficial].
    *   **Open Meshes:** Single-direction ray casting yields incorrect results for non-watertight meshes. Use multiple random directions and vote for the majority result (inside/outside) [S2, unofficial].

3.  **Interior Filling (Optional):**
    *   If `fill_interior` is true, mark all voxels determined to be "inside" by the ray cast as occupied [S3, unofficial].
    *   Alternatively, use a 3D scan-fill approach: traverse slices, toggle an inside/outside flag when crossing boundaries, and intersect results from three axis directions to eliminate singular point errors [S5, unofficial].

4.  **Attribute Assignment (SDF):**
    *   For occupied voxels, compute the distance to the nearest mesh surface point $d$ [S4, unofficial].
    *   Assign SDF: $SDF = -d$ if inside, $SDF = +d$ if outside [S4, unofficial].
    *   Apply a narrow band filter: keep only voxels where $|SDF| \leq \text{band}$ [S4, unofficial].

**Möller–Trumbore Intersection Formula:**
Given triangle vertices $V_1, V_2, V_3$, edge vectors $E_1 = V_2 - V_1$, $E_2 = V_3 - V_1$, ray origin $O$, and direction $D$:
$$ det = E_1 \cdot (D \times E_2) $$
If $|det| < \epsilon$, no intersection.
$$ inv\_det = 1 / det $$
$$ u = ((O - V_1) \times D) \cdot E_2 \cdot inv\_det $$
$$ v = (D \times (O - V_1)) \cdot E_1 \cdot inv\_det $$
$$ t = E_2 \cdot (D \times (O - V_1)) \cdot inv\_det $$
Intersection if $0 \le u \le 1$, $0 \le v \le 1$, $u+v \le 1$, and $t > \epsilon$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def voxelize_mesh(mesh_triangles, resolution, bounds_min, bounds_max, samples=1, double_thick=False, fill_interior=True):
    """
    Voxelize a mesh using ray casting.
    
    Args:
        mesh_triangles: List of tuples (v1, v2, v3), each v is (x, y, z).
        resolution: Grid dimension N (assumes cubic grid N x N x N).
        bounds_min: (x, y, z) minimum of bounding box.
        bounds_max: (x, y, z) maximum of bounding box.
        samples: Number of random rays per voxel. -1 means 1 ray along +X.
        double_thick: Boolean for double-sided mesh handling.
        fill_interior: Boolean to fill inside voxels.
        
    Returns:
        grid: 3D list of booleans.
    """
    # 1. Setup Grid
    nx, ny, nz = resolution, resolution, resolution
    dx = (bounds_max[0] - bounds_min[0]) / (nx - 2)
    dy = (bounds_max[1] - bounds_min[1]) / (ny - 2)
    dz = (bounds_max[2] - bounds_min[2]) / (nz - 2)
    
    # Initialize grid: False (empty)
    grid = [[[False for _ in range(nz)] for _ in range(ny)] for _ in range(nx)]
    
    # 2. Iterate Voxels
    for i in range(nx):
        for j in range(ny):
            for k in range(nz):
                # Compute voxel center
                cx = bounds_min[0] + (i + 0.5) * dx
                cy = bounds_min[1] + (j + 0.5) * dy
                cz = bounds_min[2] + (k + 0.5) * dz
                origin = (cx, cy, cz)
                
                is_inside = False
                
                if samples == -1:
                    # Single ray along +X
                    direction = (1.0, 0.0, 0.0)
                    hits = count_intersections(origin, direction, mesh_triangles)
                    is_inside = determine_parity(hits, double_thick)
                else:
                    # Multiple random samples
                    votes = 0
                    for _ in range(samples):
                        # Generate random direction on sphere
                        theta = random.uniform(0, 2 * math.pi)
                        z = random.uniform(-1, 1)
                        # Generate direction vector components
                        dir_x = math.sqrt(1 - z*z) * math.cos(theta)
                        dir_y = math.sqrt(1 - z*z) * math.sin(theta)
                        dir_z = math.sqrt(1 - z*z) * math.cos(theta) # Implementation detail from source [S2, unofficial]
                        direction = (dir_x, dir_y, dir_z)
                        
                        hits = count_intersections(origin, direction, mesh_triangles)
                        if determine_parity(hits, double_thick):
                            votes += 1
                    
                    # Majority vote
                    is_inside = votes > samples / 2
                
                if is_inside:
                    grid[i][j][k] = True
                    
    # 3. Interior Filling (if not already handled by parity on closed mesh)
    if fill_interior:
        # For open meshes or specific algorithms, explicit fill might be needed.
        # Here, parity on a closed mesh already fills the interior.
        pass
        
    return grid

def count_intersections(origin, direction, triangles):
    hits = []
    for v1, v2, v3 in triangles:
        hit_t = ray_triangle_intersect(origin, direction, v1, v2, v3)
        if hit_t is not None:
            hits.append(hit_t)
    
    # Sort to handle precision issues
    hits.sort()
    
    # Remove duplicate hits caused by precision (advance ray origin)
    unique_hits = []
    if hits:
        current_t = hits[0]
        unique_hits.append(current_t)
        for t in hits[1:]:
            if t > current_t + 1e-4: # Threshold to avoid re-hitting
                unique_hits.append(t)
                current_t = t
                
    return len(unique_hits)

def ray_triangle_intersect(origin, direction, v1, v2, v3):
    e1 = (v2[0]-v1[0], v2[1]-v1[1], v2[2]-v1[2])
    e2 = (v3[0]-v1[0], v3[1]-v1[1], v3[2]-v1[2])
    
    # Cross products
    # D x E2
    dxc_e2 = (direction[1]*e2[2] - direction[2]*e2[1],
              direction[2]*e2[0] - direction[0]*e2[2],
              direction[0]*e2[1] - direction[1]*e2[0])
    
    det = e1[0]*dxc_e2[0] + e1[1]*dxc_e2[1] + e1[2]*dxc_e2[2]
    
    if abs(det) < 1e-8:
        return None
        
    inv_det = 1.0 / det
    
    # O - V1
    ov1 = (origin[0]-v1[0], origin[1]-v1[1], origin[2]-v1[2])
    
    # (O-V1) x D
    ov1xd = (ov1[1]*direction[2] - ov1[2]*direction[1],
             ov1[2]*direction[0] - ov1[0]*direction[2],
             ov1[0]*direction[1] - ov1[1]*direction[0])
             
    u = (ov1xd[0]*e2[0] + ov1xd[1]*e2[1] + ov1xd[2]*e2[2]) * inv_det
    
    if u < 0.0 or u > 1.0:
        return None
        
    # D x (O-V1)
    dxd_ov1 = (direction[1]*ov1[2] - direction[2]*ov1[1],
               direction[2]*ov1[0] - direction[0]*ov1[2],
               direction[0]*ov1[1] - direction[1]*ov1[0])
               
    v = (dxd_ov1[0]*e1[0] + dxd_ov1[1]*e1[1] + dxd_ov1[2]*e1[2]) * inv_det
    
    if v < 0.0 or u + v > 1.0:
        return None
        
    # E2 x (O-V1)
    e2x_ov1 = (e2[1]*ov1[2] - e2[2]*ov1[1],
               e2[2]*ov1[0] - e2[0]*ov1[2],
               e2[0]*ov1[1] - e2[1]*ov1[0])
               
    t = (direction[0]*e2x_ov1[0] + direction[1]*e2x_ov1[1] + direction[2]*e2x_ov1[2]) * inv_det
    
    if t > 1e-8:
        return t
    return None

def determine_parity(hits, double_thick):
    if double_thick:
        return (hits // 2) % 2 == 1
    else:
        return hits % 2 == 1
```

**Edge Cases and Pitfalls:**
*   **Precision:** Ray re-hitting the same face due to floating-point error is handled by advancing the ray origin by $10^{-4}$ after each intersection or sorting and filtering hit distances [S4, unofficial].
*   **Open Meshes:** Single-direction ray casting yields incorrect results for non-watertight meshes. Use multiple random directions and voting [S2, unofficial].
*   **Double-thick Meshes:** Standard odd-even rule fails for double-sided surfaces. Use $(N_{hits}/2) \% 2 == 1$ [S2, unofficial].
*   **Malformed Input:** OBJ/STL files with non-multiple-of-3 position/index arrays must be validated and rejected [S4, unofficial].

## Subtopics

### [Solid Voxelization based on Octree](2062-solid-voxelization-based-on-octree.md)
Solid Voxelization based on Octree is a method for converting triangular surface meshes into a solid voxel representation using an adaptive octree structure. It determines the solid or empty status of leaf nodes by combining ray casting for sign determination with closest-point distance calculations. Unlike fixed-grid voxelization, this method adapts resolution to the geometry, potentially reducing node count for sparse scenes, while its reliance on ray casting for sign determination distinguishes it from exact geometric intersection methods like [Octree-based 3D Voxelization with SAT Pruning](2588-octree-based-3d-voxelization-with-sat-pruning.md).

### Solid Voxelization via 3D Scan-Fill
An extension of surface voxelization for closed objects that fills the inner region of the object using a 3D scan-filling operation [S5, unofficial]. It traverses the volume slice by slice and line by line, maintaining an inside/outside flag that toggles when the scan line spans a voxel intersecting the object boundary [S5, unofficial]. To eliminate errors from singular points, scan-filling is performed along all three axis directions, and the common voxels across all three results are retained [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N^3 \cdot T)$, where $N$ is the grid resolution per axis and $T$ is the number of triangles [G]. Each voxel center requires a ray-triangle intersection test against all triangles (or a subset via acceleration structures) [G].
*   **Memory Complexity:** Not given in the sources [G].
*   **Performance:** On an NVIDIA GTX 970, a 64x64x64 voxelization of a 69,664 triangle bunny mesh takes 0.88 seconds with 1 sample, scaling to 51.96 seconds at 256x256x256. A 512x512x512 resolution for the same mesh takes 417.62 seconds [S2, unofficial].
*   **Strengths:** Topologically robust for closed meshes; provides accurate inside/outside determination; supports SDF generation [S4, unofficial].
*   **Limitations:** Computationally expensive for high resolutions and complex meshes ($O(T)$ per voxel); sensitive to open meshes without multiple sampling; generates large files (e.g., 4.7 GB for 256^3 resolution in OBJ format) [S2, unofficial].
*   **When to use:** When topological correctness (watertightness) is critical and the mesh is closed [G].
*   **When not to use:** For very high-resolution requirements on complex meshes where rasterization-based methods are faster; for open surfaces where scan-conversion is more appropriate [G].

## Background
Mesh voxelization via ray casting is a fundamental technique in computational geometry and computer graphics, widely used in additive manufacturing (3D printing) preparation, medical imaging, and collision detection [G]. The specific CUDA-based implementation detailed in the sources was developed for MIT's 6.807 Computational Fabrication course, leveraging the Möller–Trumbore intersection algorithm for GPU acceleration [S2, unofficial]. The method has been extended to support Signed Distance Fields (SDF) and narrow-band representations for smooth surface reconstruction [S4, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] kctess5/voxelizer, https://github.com/kctess5/voxelizer, fetched 2026-10-26 (unofficial)
[S3] Guda108/3D-Voxelization-GUI, https://github.com/Guda108/3D-Voxelization-GUI, fetched 2026-10-27 (unofficial)
[S4] voxelizer-rs, https://github.com/lhemerly/voxelizer-rs, fetched 2026-10-28 (unofficial)
[S5] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
