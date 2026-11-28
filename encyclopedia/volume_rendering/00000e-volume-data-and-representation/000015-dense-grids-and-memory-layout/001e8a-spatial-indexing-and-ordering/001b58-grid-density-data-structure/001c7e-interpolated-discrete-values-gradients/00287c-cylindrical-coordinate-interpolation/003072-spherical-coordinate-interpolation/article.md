# Spherical Coordinate Interpolation

Also called: Coordinate Transformation and Interpolation, F RBaLin, F RBaRA, mip-NeRF coordinate encoding, spherical coordinate mapping

## Overview
Spherical Coordinate Interpolation is a method for reconstructing volumetric or Fourier data by mapping Cartesian sample points into spherical coordinates before sampling [S1, unofficial][S2][S3, unofficial]. It differs from its sibling, Cylindrical Coordinate Interpolation, by utilizing radial, longitudinal, and latitudinal dimensions to handle data with spherical symmetry or to optimize sampling in spherical frequency spaces [S1, unofficial]. It addresses the problem of accurately interpolating discrete polar points onto a Cartesian grid while preserving the global computational complexity of the reconstruction [S2][S3, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D Cartesian point $\mathbf{x}=(x,y,z)$ to be sampled; a boolean `triCubic` indicating interpolation order; geometric bounds `geomSize`; data grid dimensions `dataDim`; and a default `outVal` [S1, unofficial].
*   **Output:** A scalar interpolated value. If the point is out of bounds or no polar points contribute, a default `outVal` is returned [S1, unofficial][S3, unofficial].

**Data Structures**
The method assumes access to a 3D texture or grid `dataTex` with dimensions `dataDim.width`, `dataDim.height`, and `dataDim.depth` [S1, unofficial]. Geometric parameters `geomSize.x`, `geomSize.y`, and `geomSize.z` define the physical extent of the data volume [S1, unofficial].

**Steps**
1.  **Calculate Step Sizes:** Compute the sampling step sizes for length (radial), longitude, and latitude based on the geometric size and data dimensions [S1, unofficial]:
    *   `lengthStep = geomSize.x / dataDim.depth` [S1, unofficial]
    *   `longStep = geomSize.y / dataDim.width` [S1, unofficial]
    *   `latStep = 2 * geomSize.z / dataDim.height` [S1, unofficial]
2.  **Coordinate Conversion:** Convert the input Cartesian `point` to spherical coordinates $(\rho, \theta, \phi)$ using a helper function `cartToSpher` [S1, unofficial]. Add a Z-offset equal to `geomSize.z` to the resulting spherical coordinates to handle coordinate system alignment [S1, unofficial]. For local normalized depth and spherical coordinates, compute $\tilde{t} = \|\mathbf{x} - \mathbf{p}\|_2$, then $t = \frac{1}{\tilde{t}+1}$ and $\tilde{\mathbf{v}} = \left(\arccos\left(\frac{z}{\tilde{t}}\right), \arctan\left(\frac{y}{x}\right)\right)$ [S2].
3.  **Texture Coordinate Mapping:** Map the spherical coordinates to normalized texture coordinates $(u, v, w)$ by dividing by the respective step sizes [S1, unofficial]:
    *   $u = \theta / \text{longStep}$
    *   $v = \phi / \text{latStep}$
    *   $w = \rho / \text{lengthStep}$
    *   The source code assigns longitude to X, latitude to Y, and length to Z [S1, unofficial].
4.  **Boundary Check:** If any texture coordinate is outside the valid range $[0, \text{dataDim} - 1]$, return `outVal` [S1, unofficial].
5.  **Interpolation:**
    *   If `triCubic` is true, call `cubicInterpolate` [S1, unofficial].
    *   Otherwise, use linear texture sampling [S1, unofficial].

**Barycentric Interpolation Formula (F RBaLin)** [S3, unofficial]
For the specific barycentric reconstruction variant noted as F RBaLin, the method considers all polar points included within the area of the sample to be reconstructed [S3, unofficial].
1.  Map each polar point $F_{1D}(R_{\theta_t}(\rho_r))$ to a unique Cartesian pixel $(i, j)$ using rounding to the nearest integer:
    *   $i = \lfloor \rho_r \cos \theta_t \rceil$ [S3, unofficial]
    *   $j = \lfloor \rho_r \sin \theta_t \rceil$ [S3, unofficial]
2.  Compute the linear weight $\alpha_{t,r}$ for each polar point:
    $$ \alpha_{t,r} = \frac{\max(0, \frac{1}{2}\Delta p - |\rho_r \cos \theta_t - i|) \times \max(0, \frac{1}{2}\Delta p - |\rho_r \sin \theta_t - j|)}{\frac{1}{2}\Delta p \times \frac{1}{2}\Delta p} $$
    Where $\Delta p$ is the interpolation interval size [S3, unofficial].
3.  Compute the reconstructed value $F(i, j)$:
    $$ F(i, j) = \frac{\sum_{t=0}^{N_\theta-1} \sum_{r=0}^{N_\rho-1} F_{1D}(R_{\theta_t}(\rho_r)) \alpha_{t,r}}{W(i, j)} $$
    Where $W(i, j) = \sum_{t=0}^{N_\theta-1} \sum_{r=0}^{N_\rho-1} \alpha_{t,r}$ [S3, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interpolate_spherical(point, out_val, tri_cubic, geom_size, data_dim, cart_to_spher, grid):
    # 1. Calculate step sizes [S1, unofficial]
    length_step = geom_size.x / data_dim.depth
    long_step = geom_size.y / data_dim.width
    lat_step = 2.0 * geom_size.z / data_dim.height

    # 2. Convert to spherical and add Z-offset [S1, unofficial]
    spher_coord = cart_to_spher(point)
    spher_coord.z += geom_size.z

    # 3. Map to texture coordinates [S1, unofficial]
    # Mapping order from source: X=longitude, Y=latitude, Z=length
    tex_coord_x = spher_coord.y / long_step
    tex_coord_y = spher_coord.z / lat_step
    tex_coord_z = spher_coord.x / length_step

    # 4. Boundary check [S1, unofficial]
    if (tex_coord_x < 0 or tex_coord_x > data_dim.width - 1 or
        tex_coord_y < 0 or tex_coord_y > data_dim.height - 1 or
        tex_coord_z < 0 or tex_coord_z > data_dim.depth - 1):
        return out_val

    # 5. Interpolation [S1, unofficial]
    if tri_cubic:
        # Tricubic interpolation coefficients not provided in source [G]
        pass
    else:
        # Standard 3D linear interpolation [G]
        ix, iy, iz = int(tex_coord_x), int(tex_coord_y), int(tex_coord_z)
        fx, fy, fz = tex_coord_x - ix, tex_coord_y - iy, tex_coord_z - iz
        v000 = grid[ix, iy, iz]
        v100 = grid[ix+1, iy, iz]
        v010 = grid[ix, iy+1, iz]
        v110 = grid[ix+1, iy+1, iz]
        v001 = grid[ix, iy, iz+1]
        v101 = grid[ix+1, iy, iz+1]
        v011 = grid[ix, iy+1, iz+1]
        v111 = grid[ix+1, iy+1, iz+1]
        return ((1-fx)*(1-fy)*(1-fz)*v000 + fx*(1-fy)*(1-fz)*v100 +
                (1-fx)*fy*(1-fz)*v010 + fx*fy*(1-fz)*v110 +
                (1-fx)*(1-fy)*fz*v001 + fx*(1-fy)*fz*v101 +
                (1-fx)*fy*fz*v011 + fx*fy*fz*v111)
```

**Edge Cases and Pitfalls**
*   **Out of Bounds:** If texture coordinates fall outside the data dimensions, the method returns the default output value rather than clamping or wrapping [S1, unofficial].
*   **No Contribution:** In the barycentric variant, if no polar points are included in the pixel's area, the Fourier value is not reconstructed [S3, unofficial].

## Operating characteristics
*   **Complexity:** The global complexity of the reconstruction remains $O(N \log N)$, consistent with standard Fourier-based reconstruction methods [S3, unofficial].
*   **Memory:** Requires storage for the 3D data grid and geometric parameters. No additional large auxiliary structures are mentioned beyond the standard texture memory [S1, unofficial].
*   **Strengths:** Maintains low computational complexity [S3, unofficial].
*   **Limitations:** Performance depends on the efficiency of the spherical-to-Cartesian conversion and texture sampling; boundary handling is strict (no extrapolation) [S1, unofficial].

## Background
The method is described in the context of CUDA-based volume rendering [S1, unofficial] and tomographic reconstruction precision [S3, unofficial]. It is part of a family of coordinate interpolation techniques including Cylindrical Coordinate Interpolation [S1, unofficial]. The barycentric variants F RBaLin and F RBaRA are specifically discussed in the context of Fourier reconstruction quality and precision in tomography [S3, unofficial]. The periodic coordinate encoding variant is used in neural light field probe optimization [S2].

## Sources
[S1] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S2] NeLF-Pro: Neural Light Field Probes for Multi-Scale Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.01875, fetched 2026-10-27
[S3] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
