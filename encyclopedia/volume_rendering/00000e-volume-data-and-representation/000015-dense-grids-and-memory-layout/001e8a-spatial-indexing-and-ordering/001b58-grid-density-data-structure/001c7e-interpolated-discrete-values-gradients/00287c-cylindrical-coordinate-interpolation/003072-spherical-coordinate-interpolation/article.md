# Spherical Coordinate Interpolation

Also called: F RBaLin, F RBaRA

## Overview
Spherical Coordinate Interpolation is a method for reconstructing volumetric or Fourier data by mapping Cartesian sample points into spherical coordinates before sampling [S1, S2, unofficial]. It differs from its sibling, Cylindrical Coordinate Interpolation, by utilizing radial, longitudinal, and latitudinal dimensions to handle data with spherical symmetry or to optimize sampling in spherical frequency spaces [S1, unofficial]. It addresses the problem of accurately interpolating discrete polar points onto a Cartesian grid while preserving the global computational complexity of the reconstruction [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D Cartesian point `point` (x, y, z) to be sampled; a boolean `triCubic` indicating interpolation order; geometric bounds (`geomSize`); and data grid dimensions (`dataDim`).
*   **Output:** A scalar interpolated value. If the point is out of bounds or no polar points contribute, a default `outVal` is returned.

**Data Structures**
The method assumes access to a 3D texture or grid `dataTex` with dimensions `dataDim.width`, `dataDim.height`, and `dataDim.depth`. Geometric parameters `geomSize.x`, `geomSize.y`, and `geomSize.z` define the physical extent of the data volume.

**Steps**
1.  **Calculate Step Sizes:** Compute the sampling step sizes for length (radial), longitude, and latitude based on the geometric size and data dimensions [S1, unofficial].
    *   `lengthStep = geomSize.x / dataDim.depth`
    *   `longStep = geomSize.y / dataDim.width`
    *   `latStep = 2 * geomSize.z / dataDim.height`
2.  **Coordinate Conversion:** Convert the input Cartesian `point` to spherical coordinates $(\rho, \theta, \phi)$ using a helper function `cartToSpher`. Add a Z-offset equal to `geomSize.z` to the resulting spherical coordinates to handle coordinate system alignment [S1, unofficial].
3.  **Texture Coordinate Mapping:** Map the spherical coordinates to normalized texture coordinates $(u, v, w)$ by dividing by the respective step sizes [S1, unofficial]:
    *   $u = \theta / \text{longStep}$
    *   $v = \phi / \text{latStep}$
    *   $w = \rho / \text{lengthStep}$
    *   *Note:* The specific mapping order in the source code assigns longitude to X, latitude to Y, and length to Z [S1, unofficial].
4.  **Boundary Check:** If any texture coordinate is outside the valid range $[0, \text{dataDim} - 1]$, return `outVal` [S1, unofficial].
5.  **Interpolation:**
    *   If `triCubic` is true, perform trilinear or tricubic interpolation (implementation details of `cubicInterpolate` are not provided in the source, but it implies higher-order smoothing) [S1, unofficial].
    *   Otherwise, perform linear texture sampling using `tex3D` [S1, unofficial].

**Barycentric Interpolation Formula (F RBaLin)**
For the specific barycentric reconstruction variant noted as F RBaLin, the method considers all polar points included within the area of the sample to be reconstructed [S2, unofficial].
1.  Map each polar point $F_{1D}(R_{\theta_t}(\rho_r))$ to a unique Cartesian pixel $(i, j)$ using rounding to the nearest integer:
    *   $i = \lfloor \rho_r \cos \theta_t \rceil$
    *   $j = \lfloor \rho_r \sin \theta_t \rceil$
2.  Compute the linear weight $\alpha_{t,r}$ for each polar point:
    $$ \alpha_{t,r} = \frac{\max(0, \frac{1}{2}\Delta p - |\rho_r \cos \theta_t - i|) \times \max(0, \frac{1}{2}\Delta p - |\rho_r \sin \theta_t - j|)}{\frac{1}{2}\Delta p \times \frac{1}{2}\Delta p} $$
    Where $\Delta p$ is the pixel size or interpolation interval size [S2, unofficial].
3.  Compute the reconstructed value $F(i, j)$:
    $$ F(i, j) = \frac{\sum_{t=0}^{N_\theta-1} \sum_{r=0}^{N_\rho-1} F_{1D}(R_{\theta_t}(\rho_r)) \alpha_{t,r}}{W(i, j)} $$
    Where $W(i, j) = \sum_{t=0}^{N_\theta-1} \sum_{r=0}^{N_\rho-1} \alpha_{t,r}$ [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interpolate_spherical(point, out_val, tri_cubic, geom_size, data_dim, cart_to_spher, cubic_interpolate, tex3d):
    # 1. Calculate step sizes
    length_step = geom_size.x / data_dim.depth
    long_step = geom_size.y / data_dim.width
    lat_step = 2 * geom_size.z / data_dim.height

    # 2. Convert to spherical and add Z-offset
    spher_coord = cart_to_spher(point)
    spher_coord.z += geom_size.z  # Add geomSize.z offset

    # 3. Map to texture coordinates
    # Mapping: X=longitude, Y=latitude, Z=length (based on source code order)
    tex_coord_x = spher_coord.y / long_step
    tex_coord_y = spher_coord.z / lat_step
    tex_coord_z = spher_coord.x / length_step

    # 4. Boundary check
    if (tex_coord_x < 0 or tex_coord_x > data_dim.width - 1 or
        tex_coord_y < 0 or tex_coord_y > data_dim.height - 1 or
        tex_coord_z < 0 or tex_coord_z > data_dim.depth - 1):
        return out_val

    # 5. Interpolation
    if tri_cubic:
        # cubic_interpolate is assumed to be defined elsewhere or implemented as standard tricubic
        return cubic_interpolate(tex_coord_x, tex_coord_y, tex_coord_z)
    else:
        return tex3d(tex_coord_x, tex_coord_y, tex_coord_z)
```

**Edge Cases and Pitfalls**
*   **Out of Bounds:** If texture coordinates fall outside the data dimensions, the method returns the default output value rather than clamping or wrapping [S1, unofficial].
*   **No Contribution:** In the barycentric variant, if no polar points are included in the pixel's area, the Fourier value is not reconstructed (resulting in undefined or zero behavior depending on implementation) [S2, unofficial].
*   **Precision:** The use of rounding to nearest integer for pixel mapping $i, j$ can introduce quantization errors if the grid resolution is low relative to the polar point density [S2, unofficial].

## Operating characteristics
*   **Complexity:** The global complexity of the reconstruction remains $O(N \log N)$, consistent with standard Fourier-based reconstruction methods [S2, unofficial].
*   **Memory:** Requires storage for the 3D data grid and geometric parameters. No additional large auxiliary structures are mentioned beyond the standard texture memory [S1, unofficial].
*   **Strengths:** Efficiently handles data with spherical symmetry; maintains low computational complexity [S2, unofficial].
*   **Limitations:** Performance depends on the efficiency of the spherical-to-Cartesian conversion and texture sampling; boundary handling is strict (no extrapolation) [S1, unofficial].

## Background
The method is described in the context of CUDA-based volume rendering [S1, unofficial] and tomographic reconstruction precision [S2, unofficial]. It is part of a family of coordinate interpolation techniques including Cylindrical Coordinate Interpolation [S1, unofficial]. The barycentric variants F RBaLin and F RBaRA are specifically discussed in the context of Fourier reconstruction quality and precision in tomography [S2, unofficial].

## Sources
[S1] ojaisnielsen/dtu-2010-cuda-volume-rendering, https://github.com/ojaisnielsen/dtu-2010-cuda-volume-rendering, fetched 2026-10-27 (unofficial)
[S2] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
