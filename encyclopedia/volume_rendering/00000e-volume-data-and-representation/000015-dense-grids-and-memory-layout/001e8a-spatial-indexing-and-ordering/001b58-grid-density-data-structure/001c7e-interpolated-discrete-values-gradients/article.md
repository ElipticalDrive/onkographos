# Interpolated Discrete Values & Gradients

Also called: Cubic Interpolation, Cubic Spline Interpolation, CubicSpline, Discrete Field Interpolation, Interpolated Grid Fields, Tri-Cubic Interpolation

## Overview
This method computes the scalar field value at a 3D point using tri-cubic interpolation, requiring a 4×4×4 neighborhood of 64 voxels [S1]. It provides scalar value and gradient fields computed from discrete data at grid vertices using interpolation [S2]. It builds on [Grid Density Data Structure](998-grid-density-data-structure.md) and operates alongside siblings [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) and [EdgeMap Data Layout](1109-edgemap-data-layout.md) [S1, S2].

## How it works
**Inputs:**
- A query point $p = (p_x, p_y, p_z)$ in world space [S1].
- A grid structure `grid` providing spatial indexing and point access via `grid.point(i,j,k)` [S2].
- Discrete scalar values $V_{ijk}$ stored at grid vertices $(i, j, k)$ [S1, S2].

**Outputs:**
- A scalar value $f(p)$ representing the interpolated density at $p$ [S1].
- A gradient vector $\nabla f(p)$ representing the spatial derivative at $p$ [S2].

**Data Structures:**
- `Grid`: Contains bounding box bounds, voxel spacing (dimensions), and dimensions in voxels ($N_x, N_y, N_z$) [S2].
- `Values`: A 3D array or accessor mapping integer indices $(i, j, k)$ to scalar values $V_{ijk}$ [S2].

**Algorithm Steps:**

1.  **Compute voxel integer coordinates:** $v_i[\text{axis}] = \text{Pos2Voxel}(p, \text{axis})$ [S1].
2.  **Compute the minimum corner of the voxel containing $p$:** $v_{\min} = \text{bounds}[0] + v_i \times \text{voxel\_dim}$ [S1].
3.  **Compute local offset $t$ in range $[0,1]$:** $t = (p - v_{\min}) / \text{voxel\_dim}$ [S1].
4.  **Z-Interpolation:** For each $j \in [-1, 2]$ and $i \in [-1, 2]$, compute 1D cubic interpolation along $z$ using 4 values: $z\_interp[j+1][i+1] = \text{CINT\_CR}(t_z, V_{v_x+i, v_y+j, v_z-1}, V_{v_x+i, v_y+j, v_z}, V_{v_x+i, v_y+j, v_z+1}, V_{v_x+i, v_y+j, v_z+2})$ [S1].
5.  **Y-Interpolation:** Interpolate the $z$-interpolated values along $y$: $y\_interp[i] = \text{CINT\_CR}(t_y, z\_interp[0][i], z\_interp[1][i], z\_interp[2][i], z\_interp[3][i])$ for $i \in [0, 3]$ [S1].
6.  **X-Interpolation:** Interpolate the $y$-interpolated values along $x$: $f(p) = \text{CINT\_CR}(t_x, y\_interp[0], y\_interp[1], y\_interp[2], y\_interp[3])$ [S1].
7.  **Gradient Computation:** For gradients, either compute analytically per vertex or use `gradients.compute_discrete_gradients(values)` [S2].

**Cubic Interpolation Basis (Catmull-Rom):**
The function $\text{CINT\_CR}(t, p_k, p_{k+1}, m_k, m_{k+1})$ interpolates between $p_k$ and $p_{k+1}$ with derivatives $m_k$ and $m_{k+1}$ over interval $[x_k, x_{k+1}]$ [S1]. Let $\Delta = x_{k+1} - x_k$ [S1]. Let $t$ be the normalized parameter in $[0, 1]$ [S1]. Compute powers: $t_2 = t^2$, $t_3 = t^3$ [S1]. Basis functions: $h_{00} = 2t^3 - 3t^2 + 1$, $h_{10} = t^3 - 2t^2 + t$, $h_{01} = -2t^3 + 3t^2$, $h_{11} = t^3 - t^2$ [S1]. Return result: $h_{00} p_k + h_{10} \Delta m_k + h_{01} p_{k+1} + h_{11} \Delta m_{k+1}$ [S1].

**Parameters:**
- `grid`: `CGAL::Isosurfacing::Cartesian_grid_3<Kernel>` providing spatial indexing and point access [S2].
- `voxel_dim`: The physical size of a voxel (spacing) [S1].

**Edge Cases and Pitfalls:**
- Not given in the sources. [G]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def Pos2Voxel(p, axis, origin, spacing):
    """Convert world coordinate to integer voxel index [G]."""
    return int(math.floor((p[axis] - origin[axis]) / spacing[axis]))

def catmull_rom(t, pk, pk_1, mk, mk_1):
    """Catmull-Rom interpolation basis function per [S1]."""
    t2 = t * t
    t3 = t2 * t
    h00 = 2 * t3 - 3 * t2 + 1
    h10 = t3 - 2 * t2 + t
    h01 = -2 * t3 + 3 * t2
    h11 = t3 - t2
    delta = 1.0  # Unit spacing in index space [G]
    return h00 * pk + h10 * delta * mk + h01 * pk_1 + h11 * delta * mk_1

def interpolate_tricubic(p, grid, values):
    """Tri-cubic interpolation per [S1]."""
    origin = grid.bounds[0]
    spacing = grid.spacing
    
    vx = Pos2Voxel(p, 0, origin, spacing)
    vy = Pos2Voxel(p, 1, origin, spacing)
    vz = Pos2Voxel(p, 2, origin, spacing)
    
    v_min_x = origin[0] + vx * spacing[0]
    v_min_y = origin[1] + vy * spacing[1]
    v_min_z = origin[2] + vz * spacing[2]
    
    tx = (p[0] - v_min_x) / spacing[0]
    ty = (p[1] - v_min_y) / spacing[1]
    tz = (p[2] - v_min_z) / spacing[2]
    
    z_interp = [[0.0] * 4 for _ in range(4)]
    for j in range(-1, 3):
        for i in range(-1, 3):
            v0 = values(vx + i, vy + j, vz - 1)
            v1 = values(vx + i, vy + j, vz)
            v2 = values(vx + i, vy + j, vz + 1)
            v3 = values(vx + i, vy + j, vz + 2)
            # Derivatives approximated via central differences [G]
            mk = (v2 - v0) / 2.0
            mk_1 = (v3 - v1) / 2.0
            z_interp[j + 1][i + 1] = catmull_rom(tz, v1, v2, mk, mk_1)
            
    y_interp = [0.0] * 4
    for i in range(4):
        z0 = z_interp[0][i]
        z1 = z_interp[1][i]
        z2 = z_interp[2][i]
        z3 = z_interp[3][i]
        mk = (z2 - z0) / 2.0
        mk_1 = (z3 - z1) / 2.0
        y_interp[i] = catmull_rom(ty, z1, z2, mk, mk_1)
        
    y0 = y_interp[0]
    y1 = y_interp[1]
    y2 = y_interp[2]
    y3 = y_interp[3]
    mk = (y2 - y0) / 2.0
    mk_1 = (y3 - y1) / 2.0
    result = catmull_rom(tx, y1, y2, mk, mk_1)
    return result
```

## Subtopics

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
This method collects the values at the 8 vertices of the grid cell enclosing each input point, providing the raw data required for trilinear interpolation or similar operations.

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
This method retrieves the scalar density value of the single voxel containing a sample point, without interpolating neighboring values. It differs from interpolation methods by selecting only the nearest grid vertex. The "Periodic Boundary" variant handles out-of-bounds coordinates by wrapping them via modulo arithmetic, allowing seamless tiling or robust handling of rays outside the grid volume.

## Operating characteristics
- **Time and memory complexity:** Not given in the sources. [G]
- **Measured performance:** Not given in the sources. [G]
- **Scaling:** Not given in the sources. [G]
- **Strengths and limitations:** Not given in the sources. [G]
- **When to use it and when not:** Not given in the sources. [G]

## Background
The method originates from cubic spline interpolation for scalar field evaluation in volume rendering [S1]. It is implemented in CGAL's Isosurfacing module as `CGAL::Isosurfacing::Interpolated_discrete_values_3` and `CGAL::Isosurfacing::Interpolated_discrete_gradients_3`, used in `contouring_discrete_data.cpp` [S2]. It is also used in CUDA-based volume rendering projects for ray marching with transfer functions [S1].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
