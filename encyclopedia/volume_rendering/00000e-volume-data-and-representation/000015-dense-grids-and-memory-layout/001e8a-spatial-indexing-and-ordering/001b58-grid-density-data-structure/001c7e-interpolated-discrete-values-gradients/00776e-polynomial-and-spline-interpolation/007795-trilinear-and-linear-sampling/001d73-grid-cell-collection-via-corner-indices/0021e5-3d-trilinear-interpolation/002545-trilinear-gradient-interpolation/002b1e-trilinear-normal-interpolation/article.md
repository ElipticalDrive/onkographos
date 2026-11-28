# Trilinear Normal Interpolation


## Overview
Trilinear Normal Interpolation computes the surface normal at an intersection point by trilinearly interpolating pre-computed normal vectors stored at the eight corners of the intersecting voxel [S1, unofficial]. It differs from finite-difference gradient approximation by using explicitly stored vertex normals rather than deriving them on-the-fly from scalar field differences [S1, unofficial][S2].

## How it works
**Inputs:**
- `intersection_pos`: The 3D coordinates of the point where the normal is required [S1, unofficial][S2].
- `voxel_min_point`: The 3D coordinates of the minimum corner of the voxel containing `intersection_pos` [S1, unofficial].
- `voxel_size`: The uniform grid spacing along each axis [S1, unofficial].
- `grid_normal_x`, `grid_normal_y`, `grid_normal_z`: 3D arrays storing the x, y, and z components of the pre-computed normals at each grid vertex [S1, unofficial].
- `grid_res_x`, `grid_res_y`, `grid_res_z`: The dimensions of the normal grid [S1, unofficial].

**Outputs:**
- `N`: The interpolated normal vector at `intersection_pos` [S1, unofficial][S2].

**Data Structures:**
- The normal grid is stored as three separate 3D arrays (one for each component) [S1, unofficial].
- Corner normals are retrieved using integer voxel indices $(x, y, z)$ and their neighbors $(x+1, y+1, z+1)$ [S1, unofficial].

**Steps:**
1. **Compute Fractional Coordinates:** Calculate the position of the intersection point relative to the voxel minimum, normalized by the voxel size, to obtain fractional coordinates $t_x, t_y, t_z \in [0, 1]$ [S1, unofficial]:
   $$ t_x = \frac{P_x - V_{min,x}}{V_{size}} $$
   $$ t_y = \frac{P_y - V_{min,y}}{V_{size}} $$
   $$ t_z = \frac{P_z - V_{min,z}}{V_{size}} $$
   [S1, unofficial]
2. **Retrieve Corner Normals:** Identify the 8 corner normals of the voxel containing the point. Let $N_{ijk}$ be the normal at corner $(i,j,k)$ where $i,j,k \in \{0,1\}$ correspond to the lower/upper bounds in x, y, z respectively [S1, unofficial].
3. **Compute Trilinear Weights:** Calculate the weight $w_{ijk}$ for each corner $(i,j,k)$ [S1, unofficial]:
   $$ w_{ijk} = (1-t_x)^{1-i} t_x^i (1-t_y)^{1-j} t_y^j (1-t_z)^{1-k} t_z^k $$
   [S1, unofficial]
4. **Interpolate Normal:** Sum the weighted corner normals to get the interpolated vector $N_{raw}$ [S1, unofficial]:
   $$ N_{raw} = \sum_{i=0}^{1} \sum_{j=0}^{1} \sum_{k=0}^{1} w_{ijk} N_{ijk} $$
   [S1, unofficial]
5. **Normalize:** Normalize the resulting vector to ensure it has unit length [S1, unofficial][S2]:
   $$ N = \frac{N_{raw}}{\|N_{raw}\|} $$
   [S1, unofficial][S2]

**Alternative Formulation (from [S2]):**
In some contexts, the "normals" are derived from gradients of scalar amplitude data $F$ [S2]. The gradient at a grid point $(l,m,n)$ is computed using central finite differences [S2]:
$$ \nabla F_{l,m,n} = \begin{bmatrix} \frac{F_{l+1,m,n} - F_{l-1,m,n}}{2\Delta x} \\ \frac{F_{l,m+1,n} - F_{l,m-1,n}}{2\Delta y} \\ \frac{F_{l,m,n+1} - F_{l,m,n-1}}{2\Delta z} \end{bmatrix} $$
[S2]
The normal at a resampling point $S$ is then calculated by first normalizing the corner gradients, interpolating them, and normalizing the result [S2]:
$$ N(S) = \text{normalize}\left( f_{\text{Trilinear}}\left( \text{normalize}(\nabla F(S)) \right) \right) $$
[S2]
Where $f_{\text{Trilinear}}$ denotes the trilinear interpolation operation on the 8 normalized gradient vectors [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_normal_interpolation(intersection_pos, voxel_min_point, voxel_size,
                                   grid_normal_x, grid_normal_y, grid_normal_z,
                                   grid_res):
    # 1. Compute fractional coordinates within the voxel
    tx = (intersection_pos[0] - voxel_min_point[0]) / voxel_size
    ty = (intersection_pos[1] - voxel_min_point[1]) / voxel_size
    tz = (intersection_pos[2] - voxel_min_point[2]) / voxel_size

    # 2. Determine voxel indices from intersection position
    ix = int(intersection_pos[0] / voxel_size)
    iy = int(intersection_pos[1] / voxel_size)
    iz = int(intersection_pos[2] / voxel_size)

    # 3. Retrieve 8 corner normals and interpolate
    weighted_nx = 0.0
    weighted_ny = 0.0
    weighted_nz = 0.0

    for dx in (0, 1):
        for dy in (0, 1):
            for dz in (0, 1):
                # Clamp indices to grid bounds [0, res-1]
                x_idx = min(max(ix + dx, 0), grid_res[0] - 1)
                y_idx = min(max(iy + dy, 0), grid_res[1] - 1)
                z_idx = min(max(iz + dz, 0), grid_res[2] - 1)

                nx = grid_normal_x[z_idx, y_idx, x_idx]
                ny = grid_normal_y[z_idx, y_idx, x_idx]
                nz = grid_normal_z[z_idx, y_idx, x_idx]

                # Compute trilinear weight
                w = ((1 - tx)**(1 - dx) * tx**dx) * \
                    ((1 - ty)**(1 - dy) * ty**dy) * \
                    ((1 - tz)**(1 - dz) * tz**dz)

                weighted_nx += w * nx
                weighted_ny += w * ny
                weighted_nz += w * nz

    # 4. Normalize
    length = (weighted_nx**2 + weighted_ny**2 + weighted_nz**2)**0.5
    if length == 0:
        return (0.0, 0.0, 1.0) # Default normal if zero vector

    return (weighted_nx / length, weighted_ny / length, weighted_nz / length)
```

## Subtopics

### Normal Vector Calculation at Resampling Points
This variant applies trilinear normal interpolation to resampling points in volume rendering [S2]. It calculates the normal vector by interpolating the normalized gradient vectors of the eight surrounding voxel points [S2]. The process normalizes each of the eight gradient vectors, performs trilinear interpolation on them, and normalizes the resulting vector [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ per point, involving a fixed number of arithmetic operations for weights and vector summation [S1, unofficial][S2].
- **Memory Complexity:** Not given in the sources [S1, unofficial][S2].
- **Strengths:** Provides smooth, continuous normals across voxel boundaries, reducing shading artifacts [S2].
- **Limitations:** Requires pre-computation and storage of normals if not derived from scalar data [S1, unofficial]. Can suffer from precision issues if the interpolated vector is very close to zero [G].
- **When to use:** Not given in the sources [S1, unofficial][S2].
- **When not to use:** Not given in the sources [S1, unofficial][S2].

## Background
The method originates from techniques in volume rendering and computer graphics for shading discrete data [S1, unofficial][S2]. [S1, unofficial] implements it in the context of differentiable rendering (SDFDiff), where normals are needed for gradient-based optimization [S1, unofficial]. [S2] applies it to explosion field visualization to improve the Blinn-Phong lighting model's accuracy at resampling points, addressing errors from separate color and opacity interpolation [S2].

## Sources
[S1] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
[S2] A volume rendering visualization method oriented to explosion field features, https://doi.org/10.1063/5.0226283, fetched 2026-10-26
