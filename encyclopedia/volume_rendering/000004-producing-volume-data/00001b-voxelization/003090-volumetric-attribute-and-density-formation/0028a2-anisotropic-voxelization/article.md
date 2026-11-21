# Anisotropic Voxelization

Also called: Directional Voxelization, Face-based Voxelization

## Overview
Anisotropic Voxelization improves rendering quality by storing directional data (such as surface normals or radiance) within voxels, rather than averaging them into isotropic values [S1, unofficial]. It specifically solves the problem of normal cancellation in thin models (e.g., cloth) and reduces light leaking by assigning independent values to each of the six faces of a voxel cube [S1, unofficial]. Unlike isotropic methods that treat a voxel as a uniform sphere, this method distinguishes between data entering from different viewing directions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of geometric primitives (triangles) with associated attributes (e.g., color, normal vector) and a target 3D voxel grid resolution $(N_x, N_y, N_z)$.
*   **Output:** Six separate voxel grids (or a single interleaved structure), one for each of the six cardinal directions ($\pm X, \pm Y, \pm Z$).

**Data Structures:**
The method requires storing data for each face of a voxel cube. This results in 6 separate voxel grids. If the base resolution is $N_x \times N_y \times N_z$, the total storage is 6 times that of a single isotropic grid. Each voxel in a specific directional grid stores the accumulated weighted attribute for that face.

**Algorithm Steps:**
1.  **Initialize** six voxel grids, one for each axis direction: $+X, -X, +Y, -Y, +Z, -Z$. Initialize all values to zero.
2.  **Rasterize** each triangle into the voxel grid. For each triangle, determine which voxels it intersects.
3.  **For each intersected voxel** and for each of the 6 faces:
    a.  Calculate the weight for the attribute based on the face's orientation relative to the surface normal.
    b.  The weight is the dot product of the surface normal and the face's axis vector, clamped to zero.
    c.  Update the voxel's value for that face using atomic addition (averaging) of the weighted attribute.

**Formulas:**
Let $v$ be the attribute value (e.g., color or normal vector) to be stored.
Let $n$ be the surface normal of the primitive.
Let $a$ be the axis vector of the voxel face being updated. The axis vectors are:
*   $+X: (1, 0, 0)$
*   $-X: (-1, 0, 0)$
*   $+Y: (0, 1, 0)$
*   $-Y: (0, -1, 0)$
*   $+Z: (0, 0, 1)$
*   $-Z: (0, 0, -1)$

The weight $w$ for a specific face is calculated as:
$$ w = \max(a \cdot n, 0.0) $$

The value stored in the voxel for that face is updated by adding $v \times w$. If multiple primitives contribute to the same voxel face, atomic averaging is performed.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize 6 grids, each of size (Nx, Ny, Nz)
# Grids correspond to axes: +X, -X, +Y, -Y, +Z, -Z
voxel_grids = [ZeroGrid(Nx, Ny, Nz) for _ in range(6)]

# Axis vectors for the 6 faces
axes = [
    (1, 0, 0),   # +X
    (-1, 0, 0),  # -X
    (0, 1, 0),   # +Y
    (0, -1, 0),  # -Y
    (0, 0, 1),   # +Z
    (0, 0, -1)   # -Z
]

for triangle in primitives:
    # Get triangle attributes
    normal = triangle.normal
    color = triangle.color
    
    # Determine voxels intersected by the triangle
    intersected_voxels = get_intersected_voxels(triangle, grid_bounds)
    
    for voxel in intersected_voxels:
        for i, axis in enumerate(axes):
            # Calculate weight: max(axis . normal, 0.0)
            # Dot product
            dot_product = axis[0]*normal.x + axis[1]*normal.y + axis[2]*normal.z
            weight = max(dot_product, 0.0)
            
            # Weighted attribute
            weighted_color = color * weight
            
            # Atomic add to the specific face grid
            voxel_grids[i][voxel].atomic_add(weighted_color)

# Post-processing: Normalize or average the accumulated values
# Note: The source describes atomic averaging, implying division by count or weight sum
# is handled during or after the accumulation.
for i in range(6):
    normalize_grid(voxel_grids[i])
```

**Parameters:**
*   **Grid Resolution:** Not given in the sources.
*   **MSAA:** The source notes that using MSAA render targets during the voxelization process is an effective way to improve quality, particularly for thin models [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Thin Models:** In thin models (e.g., cloth), both sides of the surface may fall into the same voxel. Isotropic averaging of normals causes them to cancel out, resulting in a zero-vector and incorrect lighting. Anisotropic voxelization prevents this by storing normals per face [S1, unofficial].
*   **Light Leaking:** Isotropic voxels give a surface the same radiance in all 6 directions, potentially lighting surfaces facing away from the light source. Anisotropic voxelization reduces this by providing independent values per face [S1, unofficial].
*   **Performance:** The need for atomic averaging across 6 face grids can negatively impact the performance of the voxelization stage [S1, unofficial].
*   **Memory:** Memory usage is increased sixfold compared to isotropic voxelization [S1, unofficial].

## Subtopics

### [Anisotropic Voxelization via Projection](7835-anisotropic-voxelization-via-projection.md)
Addresses the inaccuracy of storing single density values without view direction consideration. It computes view-direction-dependent probabilities by projecting 3D voxel block grids onto 2D grids (XY, XZ, YZ planes) to produce anisotropic voxels. It sets itself apart by explicitly managing the projection of sample positions to derive axis-specific values rather than simple averaging.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. However, performance is negatively impacted due to the need for atomic averaging across 6 face grids [S1, unofficial].
*   **Memory Complexity:** Memory usage is increased sixfold compared to isotropic voxelization [S1, unofficial].
*   **Strengths:** Significantly improves final rendering quality, especially for thin models and anisotropic materials. Reduces light leaking by providing directional independence.
*   **Limitations:** High memory cost (6x). Performance overhead from atomic operations.
*   **When to use:** When rendering thin models (cloth, leaves) or materials where directional data (normals, radiance) varies significantly across the surface of a voxel.
*   **When not to use:** When memory is constrained or for homogeneous materials where directional data does not vary significantly.

## Background
Anisotropic voxelization is described in the context of real-time ray-traced voxel global illumination. It is presented as an improvement over isotropic voxelization methods to handle cases where data encoded in voxels changes significantly depending on the viewing direction, such as surface normals [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
