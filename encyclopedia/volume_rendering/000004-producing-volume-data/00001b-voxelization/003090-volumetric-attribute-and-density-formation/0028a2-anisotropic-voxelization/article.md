# Anisotropic Voxelization

Also called: Directional Voxelization, Face-based Voxelization

## Overview
Anisotropic voxelization improves rendering quality by storing directional data (such as surface normals or radiance) in separate grids for each of the six faces of a voxel cube, rather than averaging values into a single isotropic voxel [S1, unofficial]. This approach specifically addresses the cancellation of opposing surface normals in thin models (e.g., cloth) and reduces light leaking, where isotropic voxels incorrectly illuminate surfaces facing away from light sources [S1, unofficial]. It differs from [Interior Voxelize](995-interior-voxelize.md) and [Shell-based Voxelization](1748-shell-based-voxelization.md) by explicitly encoding view-dependent anisotropy at the storage level, trading a sixfold increase in memory for significantly higher fidelity in directional attributes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh or point cloud with per-vertex or per-face attributes (e.g., color, normal vector).
*   **Output:** Six 3D voxel grids (one for each axis direction: $\pm x, \pm y, \pm z$), storing weighted attribute values.

**Data Structures:**
The method requires six separate volumetric arrays. If the base resolution is $(N_x, N_y, N_z)$, the total storage is $6 \times N_x \times N_y \times N_z$ elements. Each element stores the accumulated weighted attribute (e.g., a float3 for color or normal).

**Steps:**
1.  **Initialize** six empty voxel grids corresponding to the six faces of a cube: $+x, -x, +y, -y, +z, -z$.
2.  **Rasterize** the input geometry into these grids. For each primitive (triangle/point) intersecting a voxel:
    a. Determine the surface normal $\mathbf{n}$ and the attribute value $\mathbf{v}$ (e.g., color) at the intersection.
    b. For each of the six axes $\mathbf{a} \in \{(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)\}$:
    i. Compute the weight $w = \max(\mathbf{a} \cdot \mathbf{n}, 0.0)$.
    ii. Accumulate the weighted value into the corresponding voxel face grid: $\text{Grid}_{\mathbf{a}}[voxel] \leftarrow \text{Grid}_{\mathbf{a}}[voxel] + \mathbf{v} \cdot w$.
    iii. Accumulate the weight for normalization: $\text{WeightGrid}_{\mathbf{a}}[voxel] \leftarrow \text{WeightGrid}_{\mathbf{a}}[voxel] + w$.
3.  **Normalize** (optional but typical): Divide the accumulated attribute by the accumulated weight in each grid to prevent bias from high-weight intersections.

**Formulas:**
The weight for storing a value $\mathbf{v}$ into a specific voxel face defined by axis vector $\mathbf{a}$ is:
$$ w = \max(\mathbf{a} \cdot \mathbf{n}, 0.0) $$
For example, for the negative x-face ($\mathbf{a} = (-1, 0, 0)$):
$$ w = \max(-n_x, 0.0) $$
The stored value $\mathbf{v}_{stored}$ is:
$$ \mathbf{v}_{stored} = \mathbf{v} \cdot w $$

**Parameters:**
*   **Grid Count:** Fixed at 6.
*   **Weighting Function:** Clamped dot product $\max(\mathbf{a} \cdot \mathbf{n}, 0.0)$.

**Edge Cases and Pitfalls:**
*   **Thin Models:** In isotropic voxelization, normals from opposite sides of a thin surface (like cloth) average to zero, causing lighting failures. Anisotropic voxelization stores them in separate face grids, preserving directionality [S1, unofficial].
*   **Light Leaking:** Isotropic voxels give the same radiance in all directions, lighting surfaces facing away from lights. Anisotropic voxelization restricts radiance to the relevant face, reducing this artifact [S1, unofficial].
*   **Atomic Operations:** The accumulation step requires atomic averaging across the 6 face grids, which can negatively impact performance on GPU hardware [S1, unofficial].
*   **Memory Usage:** Memory consumption is six times that of a standard isotropic voxel grid [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize 6 grids for +x, -x, +y, -y, +z, -z
# Each grid is a 3D array of shape (Nx, Ny, Nz)
# Grids store accumulated weighted values (e.g., float3)
# WeightGrids store accumulated weights (float)
grids = [EmptyGrid(Nx, Ny, Nz) for _ in range(6)]
weight_grids = [EmptyFloatGrid(Nx, Ny, Nz) for _ in range(6)]

# Axis vectors for the 6 faces
axes = [
    (1, 0, 0),   # +x
    (-1, 0, 0),  # -x
    (0, 1, 0),   # +y
    (0, -1, 0),  # -y
    (0, 0, 1),   # +z
    (0, 0, -1)   # -z
]

for triangle in mesh:
    # Compute intersection with voxel grid
    intersections = intersect_triangle_with_voxels(triangle)
    
    for voxel_pos, normal, color in intersections:
        for i, axis in enumerate(axes):
            # Compute weight: max(dot(axis, normal), 0.0)
            # axis is a tuple, normal is a vector
            dot_product = axis[0]*normal.x + axis[1]*normal.y + axis[2]*normal.z
            weight = max(dot_product, 0.0)
            
            # Accumulate weighted color
            # grids[i] is the grid for this axis
            grids[i][voxel_pos] += color * weight
            
            # Accumulate weight for normalization
            weight_grids[i][voxel_pos] += weight

# Normalize grids
for i in range(6):
    for voxel_pos in all_voxels:
        w = weight_grids[i][voxel_pos]
        if w > 0:
            grids[i][voxel_pos] /= w
```

## Subtopics

### Anisotropic Voxelization via Projection
This variant computes view-direction-dependent probability of events using finer resolution voxel data (level 3) to produce anisotropic voxels, addressing the inaccuracy of storing single density values without view direction consideration [S2]. It projects 3D voxel block grids to 2D planes (XY, XZ, YZ) using GPU shared memory, storing the projected position of every sample by block to obtain a value per axis [S2]. Occupancy $O$ for second-level voxels is computed by analyzing third-level data: dividing the number of different identifiers by the amount of voxels in the third level of voxelization [S2]. This method is chosen when correlated media or microgeometry require density estimation that varies significantly with view direction [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot F)$, where $N$ is the number of primitives and $F=6$ is the number of face grids. The factor of 6 is constant but increases the constant factor significantly.
*   **Memory Complexity:** $6 \times N_x \times N_y \times N_z$ elements. This is a sixfold increase over isotropic voxelization [S1, unofficial].
*   **Performance:** The need for atomic operations across 6 grids can negatively impact voxelization stage performance, particularly on GPUs where atomic contention is high [S1, unofficial].
*   **Strengths:** Eliminates normal cancellation in thin models; significantly reduces light leaking; improves rendering quality for anisotropic materials [S1, unofficial].
*   **Limitations:** High memory footprint; slower voxelization due to atomic averaging; not suitable for memory-constrained environments [S1, unofficial].
*   **When to use:** When rendering thin geometry (cloth, leaves) or when light leaking is a critical issue in real-time global illumination [S1, unofficial].
*   **When not to use:** When memory is strictly limited or when the scene contains only thick, homogeneous volumes where isotropic approximation is sufficient [S1, unofficial].

## Background
Anisotropic voxelization was introduced as a method to improve the quality of voxel-based global illumination by addressing the limitations of isotropic data storage [S1, unofficial]. It was notably discussed in the context of real-time ray-traced voxel global illumination, where it was shown to be an effective way to handle directional data like surface normals [S1, S1, unofficial]. The technique is particularly relevant for scenes with thin geometry and correlated media, where view-dependent density and normal estimation are crucial for accurate rendering [S2].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Fast Voxelization and Level of Detail for Microgeometry Rendering, https://doi.org/10.48550/arxiv.2604.13191, fetched 2026-10-26
