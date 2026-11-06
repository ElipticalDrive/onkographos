# Interpolated Discrete Values & Gradients

Also called: Cubic Spline Interpolation, CubicSpline, Discrete Field Interpolation, Interpolated Grid Fields, Tri-Cubic Interpolation

## Overview
Interpolated Discrete Values & Gradients computes a continuous scalar field and its spatial gradient at arbitrary 3D query points by interpolating discrete values stored at grid vertices using tri-cubic (trilinear or cubic spline) interpolation [S1, unofficial][S2]. It solves the problem of mapping continuous ray-marching or surface-extraction coordinates to the discrete tensor indices required for dense volume rendering, providing smooth density estimates and derivative information for shading or isosurfacing [S1, unofficial][S2]. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md) which yields discontinuous blocky approximations, this method blends adjacent voxels to ensure $C^0$ or $C^1$ continuity [S5]. It builds on [Grid Density Data Structure](998-grid-density-data-structure.md) by providing the sampling primitive that accesses the underlying grid data.

## How it works
**Inputs:**
- `p`: Query point in continuous world/voxel space ($\mathbb{R}^3$).
- `grid`: A 3D structured grid defining the spatial domain, spacing (`voxel_dim`), and bounds (`bounds`) [S1, unofficial][S2].
- `data`: A 3D array (tensor) of scalar values stored at grid vertices, indexed by integer coordinates $(i, j, k)$ [S1, unofficial][S2].

**Outputs:**
- `value`: The interpolated scalar field value at `p` [S1, unofficial][S2].
- `gradient`: The interpolated gradient vector $\nabla f(p)$ at `p` [S2]. If analytical gradients are unavailable, they are computed from discrete values using finite differences [S2].

**Data Structures:**
- `Grid`: Contains `bounds` (min/max coordinates), `spacing` (voxel dimensions $dx, dy, dz$), and dimensions `xdim`, `ydim`, `zdim` [S2].
- `Values`: A 3D array of size `xdim` $\times$ `ydim` $\times$ `zdim` storing scalar values at vertices [S2].
- `Gradients`: A 3D array of size `xdim` $\times$ `ydim` $\times$ `zdim` storing 3D vectors at vertices, or computed on-the-fly [S2].

**Algorithm Steps:**

1. **Voxel Coordinate Calculation:**
   Determine the integer coordinates of the voxel containing point `p`.
   $$ v_i[axis] = \lfloor \frac{p[axis] - bounds[0][axis]}{voxel\_dim[axis]} \rfloor $$
   where $axis \in \{x, y, z\}$ [S1, unofficial].

2. **Local Offset Calculation:**
   Compute the minimum corner of the voxel containing `p`:
   $$ v_{min}[axis] = bounds[0][axis] + v_i[axis] \times voxel\_dim[axis] $$
   Compute the normalized fractional offset $t$ in range $[0, 1]$:
   $$ t[axis] = \frac{p[axis] - v_{min}[axis]}{voxel\_dim[axis]} $$
   [S1, unofficial].

3. **Tri-Cubic Interpolation (Value):**
   The method uses cubic spline interpolation along each axis, requiring a $4 \times 4 \times 4$ neighborhood of voxels (indices from $v_i[axis]-1$ to $v_i[axis]+2$) [S1, unofficial].
   
   a. **Z-Interpolation:** For each $j \in [-1, 2]$ and $i \in [-1, 2]$, compute 1D cubic interpolation along the z-axis using 4 values:
   $$ z\_interp[j+1][i+1] = \text{CINT\_CR}(t.z, \text{data}[v_i.x+i, v_i.y+j, v_i.z-1], \text{data}[v_i.x+i, v_i.y+j, v_i.z], \text{data}[v_i.x+i, v_i.y+j, v_i.z+1], \text{data}[v_i.x+i, v_i.y+j, v_i.z+2]) $$
   [S1, unofficial].

   b. **Y-Interpolation:** For each $i \in [-1, 2]$, interpolate the z-interpolated values along the y-axis:
   $$ y\_interp[i] = \text{CINT\_CR}(t.y, z\_interp[0][i], z\_interp[1][i], z\_interp[2][i], z\_interp[3][i]) $$
   [S1, unofficial].

   c. **X-Interpolation:** Interpolate the y-interpolated values along the x-axis to get the final result:
   $$ value = \text{CINT\_CR}(t.x, y\_interp[0], y\_interp[1], y\_interp[2], y\_interp[3]) $$
   [S1, unofficial].

4. **Gradient Computation:**
   Gradients can be computed analytically per vertex if available, or derived from discrete values using finite differences [S2].
   - **Analytical:** If the underlying function is known, evaluate its derivative at the grid vertices [S2].
   - **Discrete (Finite Difference):** Use `gradients.compute_discrete_gradients(values)` to approximate gradients from the `Values` field [S2].
   - **Interpolated Gradient:** Once vertex gradients are established, interpolate them at `p` using the same tri-cubic basis functions as the scalar value, or use the derivative of the interpolation basis functions directly [S2].

**Cubic Spline Basis Function (CINT_CR):**
The 1D cubic interpolation function `CINT_CR` (Cubic Interpolation with Continuous Derivatives) for points $p_k, p_{k+1}$ with derivatives $m_k, m_{k+1}$ over interval $[x_k, x_{k+1}]$ is defined as [S1, unofficial]:
$$ \delta = x_{k+1} - x_k $$
$$ t = \frac{\max(x - x_k, 0)}{\delta} $$
$$ t_2 = t^2, \quad t_3 = t^3 $$
$$ h_{00} = 2t^3 - 3t^2 + 1 $$
$$ h_{10} = t^3 - 2t^2 + t $$
$$ h_{01} = -2t^3 + 3t^2 $$
$$ h_{11} = t^3 - t^2 $$
$$ \text{Result} = h_{00} p_k + h_{10} \delta m_k + h_{01} p_{k+1} + h_{11} \delta m_{k+1} $$
[S1, unofficial].

**Parameters:**
- `grid`: The spatial indexing structure, e.g., `CGAL::Isosurfacing::Cartesian_grid_3<Kernel>` [S2].
- `isovalue`: Used in isosurfacing contexts (e.g., Marching Cubes, Dual Contouring) to determine surface extraction, not interpolation itself [S2].

**Edge Cases and Pitfalls:**
- **Boundary Access:** The $4 \times 4 \times 4$ neighborhood requires accessing indices outside the grid bounds. Standard practice is to clamp indices to the grid boundaries or use periodic boundary conditions if applicable [S1, unofficial][S2].
- **Precision:** Floating-point precision in offset calculation $t$ can affect smoothness. Use double precision for high-resolution grids [S1, unofficial].
- **Derivative Continuity:** The cubic spline ensures $C^1$ continuity if derivatives are consistent. Inconsistent derivatives (e.g., from noisy data) can cause artifacts [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_spline_1d(t, p0, p1, m0, m1):
    """
    1D cubic interpolation with derivatives.
    t: normalized parameter in [0, 1]
    p0, p1: values at endpoints
    m0, m1: derivatives at endpoints (scaled by interval length if necessary)
    """
    t2 = t * t
    t3 = t2 * t
    h00 = 2 * t3 - 3 * t2 + 1
    h10 = t3 - 2 * t2 + t
    h01 = -2 * t3 + 3 * t2
    h11 = t3 - t2
    return h00 * p0 + h10 * m0 + h01 * p1 + h11 * m1

def interpolate_tricubic(p, grid, data):
    """
    Interpolate scalar value at point p using tri-cubic interpolation.
    p: tuple (x, y, z)
    grid: object with bounds, spacing, xdim, ydim, zdim
    data: 3D array of values
    """
    # 1. Voxel coordinates
    vx = int((p[0] - grid.bounds[0][0]) / grid.spacing[0])
    vy = int((p[1] - grid.bounds[0][1]) / grid.spacing[1])
    vz = int((p[2] - grid.bounds[0][2]) / grid.spacing[2])
    
    # Clamp indices to valid range [0, dim-1] for boundary handling
    vx = max(0, min(vx, grid.xdim - 1))
    vy = max(0, min(vy, grid.ydim - 1))
    vz = max(0, min(vz, grid.zdim - 1))
    
    # 2. Local offset
    vmin_x = grid.bounds[0][0] + vx * grid.spacing[0]
    vmin_y = grid.bounds[0][1] + vy * grid.spacing[1]
    vmin_z = grid.bounds[0][2] + vz * grid.spacing[2]
    
    tx = (p[0] - vmin_x) / grid.spacing[0]
    ty = (p[1] - vmin_y) / grid.spacing[1]
    tz = (p[2] - vmin_z) / grid.spacing[2]
    
    # 3. Tri-cubic interpolation
    # Z-interpolation: 4x4 grid of values
    z_interp = [[0.0 for _ in range(4)] for _ in range(4)]
    for j in range(-1, 3):
        for i in range(-1, 3):
            # Access data with boundary clamping
            idx_x = max(0, min(vx + i, grid.xdim - 1))
            idx_y = max(0, min(vy + j, grid.ydim - 1))
            idx_z = max(0, min(vz + k, grid.zdim - 1)) # k loop below
            # Note: CINT_CR requires 4 points along Z
            # We need data[vx+i, vy+j, vz-1] to data[vx+i, vy+j, vz+2]
            vals_z = []
            for k in range(-1, 3):
                idx_z_k = max(0, min(vz + k, grid.zdim - 1))
                vals_z.append(data[idx_x][idx_y][idx_z_k])
            z_interp[j+1][i+1] = cubic_spline_1d(tz, vals_z[0], vals_z[1], vals_z[2], vals_z[3])
            
    # Y-interpolation: 4 values
    y_interp = [0.0 for _ in range(4)]
    for i in range(-1, 3):
        vals_y = [z_interp[j+1][i+1] for j in range(-1, 3)]
        y_interp[i+1] = cubic_spline_1d(ty, vals_y[0], vals_y[1], vals_y[2], vals_y[3])
        
    # X-interpolation: 1 value
    vals_x = y_interp
    result = cubic_spline_1d(tx, vals_x[0], vals_x[1], vals_x[2], vals_x[3])
    
    return result
```

## Subtopics

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Grid Cell Collection via Corner Indices is a sampling strategy that extracts the values and positions of the eight vertices defining the grid cell enclosing a set of input points. It serves as the foundational data retrieval step for trilinear interpolation and surface extraction methods, solving the problem of mapping arbitrary continuous coordinates to the discrete tensor indices required for dense volume rendering. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md) which selects a single representative value, or [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) which uses derivative information, this method explicitly gathers the full local neighborhood to enable higher-order reconstruction.

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
Nearest Voxel Lookup retrieves the scalar value of the single voxel containing a sample point without interpolating neighboring values. It maps continuous sample coordinates to discrete grid indices, applying periodic boundary conditions (modulo arithmetic) to handle out-of-bounds access. Unlike [Interpolated Discrete Values & Gradients](1268-interpolated-discrete-values-gradients.md), which blends adjacent voxels, this method provides a discontinuous, blocky approximation that is computationally cheaper but less smooth.

### [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md)
This method interpolates a normalized scalar field within a 1D interval using two piecewise quadratic Bézier curves to satisfy Hermite data (value and derivative) at both endpoints. It solves the incompatibility of single quadratic interpolation with four constraints (two values, two derivatives) by splitting the interval, providing exact interpolation in specific cases and better stability than cubic interpolation for small subdivision counts.

### [Cylindrical Coordinate Interpolation](4282-cylindrical-coordinate-interpolation.md)
Cylindrical Coordinate Interpolation maps Cartesian query points to cylindrical space before sampling the volume data, enabling efficient texture-based lookup for geometries with cylindrical symmetry. It differs from standard Cartesian interpolation by remapping the coordinate system to align with the geometry's principal axis, reducing sampling artifacts for radially symmetric data.

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Dense Copy Extraction extracts a region defined by a bounding box into a dense 3D array, filled with the grid's background value, then overwritten with `get_value` for every coordinate in the box. It calculates dimensions `dx, dy, dz` from `bbox`, initializes `result` array filled with `tree.background`, and uses a triple nested loop over `ix, iy, iz` to compute coordinates and set `result[ix + 1, iy + 1, iz + 1] = get_value(tree, c)`.

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Mipmap Grid Query queries grid values at given points using a mipmapping strategy based on the distance from the center of the base AABB, assuming AABBs of multiple grids are 2x scaled. It normalizes points to [0, 1], computes maxval, determines mip level using frexp exponent, re-normalizes points, maps to grid indices, and returns queried values multiplied by a selector.

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Uncertain Gradient Field Interpolation reconstructs the gradient field from an uncertain scalar field, resulting in a trivariate random vector Y representing uncertainties of partial derivatives. It assigns zero weights to voxels outside the support of $\rho$, involving only M neighbors. The gradient is computed as $\begin{bmatrix} Y_x \\ Y_y \\ Y_z \end{bmatrix} = \sum_i \begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} X_i$ with weights $\begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} = \rho(\mathbf{v} - \mathbf{v}_i)$, where $\rho$ is the basis function (derivative reconstruction filter).

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Backprojection via Shepard's Interpolation fits a continuous scale field S(x) from a discrete set of LoG extrema N={(xi,ti)} using basis functions, allowing for a smooth transfer function. It uses the fourth-degree Wendland polynomial $\Theta(d) = [1 - d/h]_0^1)^4 (4d/h + 1)$ for blending. Alternatively, it can use the maximum operator $S(x) = \max_{x_i \in N} {\Theta(||x - x_i||) t_i}$ to prioritize larger features. Parameter $h$ controls local support, typically $h = k \cdot t_i$ with $k=1$. Pitfall: Bleeding, where small features inherit large scales, handled by visibility tests.

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, as it involves a fixed number of arithmetic operations and array lookups (64 values for tri-cubic) [S1, unofficial].
- **Memory Complexity:** $O(N)$ where $N$ is the number of grid vertices, to store the `data` and optional `gradients` arrays [S2].
- **Measured Performance:** Performance depends on cache efficiency of accessing the $4 \times 4 \times 4$ neighborhood. On modern GPUs, coalesced memory access is critical [S1, unofficial].
- **Scaling:** Scales linearly with the number of query points. Higher grid resolution increases memory usage but not per-query cost [S1, unofficial].
- **Strengths:** Smooth interpolation ($C^1$ continuity with cubic splines), accurate gradient estimation for shading and isosurfacing [S1, unofficial][S2].
- **Limitations:** Higher computational cost than nearest-neighbor lookup. Boundary handling can introduce artifacts if not managed carefully [S1, unofficial][S5].
- **When to Use:** When smooth density fields and accurate gradients are required for volume rendering, ray marching, or isosurface extraction [S1, unofficial][S2].
- **When Not to Use:** When performance is critical and smoothness is not required (use [Nearest Voxel Lookup](1573-nearest-voxel-lookup-with-periodic-boundary.md)), or when data is sparse (use sparse grid structures) [S5].

## Background
The method originates from computer graphics and scientific visualization, where continuous fields are sampled from discrete grids for rendering and analysis [S1, unofficial]. It is widely used in isosurfacing algorithms like Marching Cubes and Dual Contouring, where smooth interpolation of scalar values and gradients is essential for accurate surface reconstruction [S2]. The use of cubic splines for interpolation dates back to early computer graphics literature, with implementations in libraries like CGAL providing robust, generic interfaces for 3D isosurfacing [S2]. The extension to uncertain gradient fields and Shepard's interpolation for transfer functions reflects ongoing research into robust and intuitive volume exploration techniques [S5][S6, unofficial].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S3] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S4] nerfstudio-project/nerfacc, https://github.com/nerfstudio-project/nerfacc, fetched 2026-10-26 (unofficial)
[S5] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
[S6] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
