# MERF Scene Representation
Also called: MERF Grid Representation, MERF Representation, Memory-Efficient Radiance Field

## Overview
MERF is a radiance field representation designed for real-time volumetric rendering of large-scale, unbounded scenes with a low memory footprint [S1]. It achieves this by combining a sparse low-resolution 3D voxel grid with three high-resolution 2D feature planes, storing feature vectors that are summed and nonlinearly mapped to density, color, and view-dependent features [S1][S2, abstract only, unofficial]. Unlike standard dense grids which scale cubically, MERF’s hybrid structure allows high-frequency detail via the 2D planes while maintaining global structure in the 3D grid, and it employs a piecewise-projective contraction to handle unbounded space efficiently [S1][S2, abstract only, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D query location $\mathbf{x} = (x, y, z)$ along a ray.
*   **Output:** Density $\tau \in \mathbb{R}^+$, diffuse RGB color $\mathbf{c}_d \in \mathbb{R}^3$, and view-dependent feature vector $\mathbf{f} \in \mathbb{R}^{d_f}$.

**Data Structures**
The representation consists of two types of grids, each storing vectors of dimension $d = 4 + d_f$:
1.  **3D Voxel Grid ($V$):** A low-resolution grid of size $N_v \times N_v \times N_v$.
2.  **2D Feature Planes ($P_x, P_y, P_z$):** Three high-resolution grids, one for each cardinal plane (XY, YZ, ZX). Each has size $N_h \times N_h$.
    *   $P_x$ corresponds to the plane perpendicular to the X-axis (YZ plane).
    *   $P_y$ corresponds to the plane perpendicular to the Y-axis (XZ plane).
    *   $P_z$ corresponds to the plane perpendicular to the Z-axis (XY plane).
    *   *Note on indexing:* The source defines $P_x: \mathbb{R}^2 \to \mathbb{R}^d$ as the grid perpendicular to the x-axis [S1].

**Parameters**
*   $N_v$: Low-resolution 3D grid dimension. Default: 512 [S1].
*   $N_h$: High-resolution 2D grid dimension. Default: 2048 [S1].
*   $d_f$: View-dependent feature dimension. Default: 8 [S1].
*   $d$: Total channels per element. $d = 4 + d_f = 12$ by default [S1].
*   Contraction: Piecewise-projective contraction is used to map unbounded scene coordinates into a bounded volume, enabling efficient ray-box intersection for empty space skipping [S1][S2, abstract only, unofficial].

**Algorithm**
To evaluate the radiance field at location $\mathbf{x}$:

1.  **Query 3D Grid:** Retrieve the 8 corner values of the low-resolution 3D grid $V$ surrounding $\mathbf{x}$.
2.  **Query 2D Planes:** Project $\mathbf{x}$ onto the three cardinal planes to obtain 2D coordinates $(y, z)$, $(x, z)$, and $(x, y)$. For each plane, retrieve the 4 corner values of the high-resolution 2D grid surrounding the projected point.
3.  **Interpolate:**
    *   Trilinearly interpolate the 8 values from $V$ to get $V(\mathbf{x})$.
    *   Bilinearly interpolate the 4 values from $P_x$ to get $P_x(y, z)$.
    *   Bilinearly interpolate the 4 values from $P_y$ to get $P_y(x, z)$.
    *   Bilinearly interpolate the 4 values from $P_z$ to get $P_z(x, y)$.
4.  **Sum Features:** Compute the raw feature vector $\mathbf{t}$ by summing the interpolated components:
    $$ \mathbf{t}(x, y, z) = V(x, y, z) + P_x(y, z) + P_y(x, z) + P_z(x, y) $$
    where $V: \mathbb{R}^3 \to \mathbb{R}^d$ is the trilinear interpolation operator and $P_{axis}: \mathbb{R}^2 \to \mathbb{R}^d$ is the bilinear interpolation operator [S1].
5.  **Split and Nonlinear Map:** Split $\mathbf{t}$ into three components: raw density $\tilde{\tau}$, raw color $\tilde{\mathbf{c}}_d$, and raw features $\tilde{\mathbf{f}}$. Apply nonlinearities to obtain the final values [S1]:
    $$ \tau = \exp(\tilde{\tau}) $$
    $$ \mathbf{c}_d = \sigma(\tilde{\mathbf{c}}_d) $$
    $$ \mathbf{f} = \sigma(\tilde{\mathbf{f}}) $$
    where $\sigma$ is the standard logistic sigmoid function, constraining colors and features to $(0, 1)$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_merf(x, y, z, V, Px, Py, Pz, Nv, Nh, d_f):
    """
    Query MERF representation at 3D location (x, y, z).
    
    Args:
        x, y, z: Float coordinates in the contracted space.
        V: 3D grid of shape (Nv, Nv, Nv, d), where d = 4 + d_f.
        Px: 2D grid of shape (Nh, Nh, d) for YZ plane.
        Py: 2D grid of shape (Nh, Nh, d) for XZ plane.
        Pz: 2D grid of shape (Nh, Nh, d) for XY plane.
        Nv: Integer, resolution of 3D grid.
        Nh: Integer, resolution of 2D grids.
        d_f: Integer, dimension of view-dependent features.
        
    Returns:
        tau: Float, density.
        cd: Array, diffuse RGB color.
        f: Array, view-dependent features.
    """
    d = 4 + d_f
    
    # 1. Query and Trilinearly Interpolate 3D Grid V
    # Get grid indices
    ix, iy, iz = x * (Nv - 1), y * (Nv - 1), z * (Nv - 1)
    # Get fractional parts
    fx, fy, fz = ix - floor(ix), iy - floor(iy), iz - floor(iz)
    # Get integer indices for corners
    x0, y0, z0 = floor(ix), floor(iy), floor(iz)
    x1, y1, z1 = x0 + 1, y0 + 1, z1 + 1 # Note: z1 is iz+1
    
    # Handle boundary conditions if necessary [G]
    # Retrieve 8 values
    v000 = V[x0, y0, z0]
    v100 = V[x1, y0, z0]
    v010 = V[x0, y1, z0]
    v110 = V[x1, y1, z0]
    v001 = V[x0, y0, z1]
    v101 = V[x1, y0, z1]
    v011 = V[x0, y1, z1]
    v111 = V[x1, y1, z1]
    
    # Trilinear interpolation
    # Interpolate along x
    v00 = lerp(v000, v100, fx)
    v01 = lerp(v010, v110, fx)
    v10 = lerp(v001, v101, fx)
    v11 = lerp(v011, v111, fx)
    # Interpolate along y
    v0 = lerp(v00, v01, fy)
    v1 = lerp(v10, v11, fy)
    # Interpolate along z
    V_interp = lerp(v0, v1, fz)
    
    # 2. Query and Bilinearly Interpolate 2D Planes
    
    # Px (YZ plane): coordinates (y, z)
    py, pz = y * (Nh - 1), z * (Nh - 1)
    fpy, fpz = py - floor(py), pz - floor(pz)
    y0_p, z0_p = floor(py), floor(pz)
    y1_p, z1_p = y0_p + 1, z0_p + 1
    
    p00 = Px[y0_p, z0_p]
    p10 = Px[y1_p, z0_p]
    p01 = Px[y0_p, z1_p]
    p11 = Px[y1_p, z1_p]
    
    # Bilinear interpolation for Px
    p0 = lerp(p00, p10, fpy)
    p1 = lerp(p01, p11, fpy)
    Px_interp = lerp(p0, p1, fpz)
    
    # Py (XZ plane): coordinates (x, z)
    px, pz = x * (Nh - 1), z * (Nh - 1)
    fpx, fpz = px - floor(px), pz - floor(pz)
    x0_p, z0_p = floor(px), floor(pz)
    x1_p, z1_p = x0_p + 1, z0_p + 1
    
    p00 = Py[x0_p, z0_p]
    p10 = Py[x1_p, z0_p]
    p01 = Py[x0_p, z1_p]
    p11 = Py[x1_p, z1_p]
    
    # Bilinear interpolation for Py
    p0 = lerp(p00, p10, fpx)
    p1 = lerp(p01, p11, fpx)
    Py_interp = lerp(p0, p1, fpz)
    
    # Pz (XY plane): coordinates (x, y)
    px, py = x * (Nh - 1), y * (Nh - 1)
    fpx, fpy = px - floor(px), py - floor(py)
    x0_p, y0_p = floor(px), floor(py)
    x1_p, y1_p = x0_p + 1, y0_p + 1
    
    p00 = Pz[x0_p, y0_p]
    p10 = Pz[x1_p, y0_p]
    p01 = Pz[x0_p, y1_p]
    p11 = Pz[x1_p, y1_p]
    
    # Bilinear interpolation for Pz
    p0 = lerp(p00, p10, fpx)
    p1 = lerp(p01, p11, fpx)
    Pz_interp = lerp(p0, p1, fpy)
    
    # 3. Sum Features
    t = V_interp + Px_interp + Py_interp + Pz_interp
    
    # 4. Split and Nonlinear Map
    # t has shape (d,)
    # First element is raw density, next 3 are raw color, rest are features
    tau = exp(t[0])
    cd = sigmoid(t[1:4])
    f = sigmoid(t[4:])
    
    return tau, cd, f

def lerp(a, b, t):
    return a + t * (b - a)

def sigmoid(x):
    return 1.0 / (1.0 + exp(-x))
```

**Edge Cases and Pitfalls**
*   **Boundary Conditions:** The pseudocode assumes coordinates are within $[0, 1]$ after contraction. If coordinates fall outside, clamping or wrapping must be applied [G].
*   **Precision:** Trilinear and bilinear interpolation require careful handling of floating-point precision, especially with high-resolution 2D grids ($N_h=2048$) [S1].
*   **Contraction:** The piecewise-projective contraction maps unbounded space to a bounded volume. Rays must be intersected with the contracted bounding box before querying the grids [S1][S2, abstract only, unofficial].

## Subtopics

### SNeRG Deferred Shading
MERF uses the deferred shading model from SNeRG, where density, color, and features are accumulated along the ray [S1].

### Piecewise-Projective Contraction
A novel contraction function that maps straight lines to a small number of line segments, enabling efficient ray-box intersection for empty space skipping, unlike spherical contraction which maps lines to curves [S1][S2, abstract only, unofficial].

## Operating characteristics
*   **Memory Complexity:** The memory footprint is dominated by the 2D grids. Total memory is proportional to $N_v^3 \cdot d + 3 \cdot N_h^2 \cdot d$. With defaults ($N_v=512, N_h=2048, d=12$), this is significantly less than a dense 3D grid of resolution $N_h^3$ [S1].
*   **Time Complexity:** Querying one point involves 8 trilinear interpolations and 12 bilinear interpolations (4 per plane). This is constant time per query, independent of scene size, enabling real-time rendering [S1][S2, abstract only, unofficial].
*   **Performance:** Enables real-time rendering of large-scale unbounded scenes in a browser [S2, abstract only, unofficial].
*   **Strengths:** Low memory consumption, real-time performance, handles unbounded scenes efficiently [S1][S2, abstract only, unofficial].
*   **Limitations:** Representational power depends on the grid resolutions; very fine details may require higher $N_h$ or $N_v$ [S1].

## Background
MERF was introduced by Hedman et al. in 2022 [S1]. It builds on the SNeRG deferred shading model [S1]. The representation was designed to address the trade-off between memory consumption and rendering speed in radiance fields, enabling real-time view synthesis of large-scale scenes [S1][S2, abstract only, unofficial].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
[S2] MERF, https://creiser.github.io/merf/, fetched 2026-10-29 (abstract only, unofficial)
