# Interpolated Discrete Values & Gradients

Also called: Cubic Spline Interpolation, CubicSpline, Discrete Field Interpolation, Interpolated Grid Fields, Tri-Cubic Interpolation

## Overview
Interpolated Discrete Values & Gradients computes a continuous scalar field and its spatial gradient at arbitrary 3D points by applying tri-cubic interpolation to discrete data stored on a dense grid [S1, unofficial][S2]. It solves the problem of mapping continuous world-space coordinates to discrete memory indices while providing $C^1$-continuous derivatives, which are essential for accurate surface normal estimation in isosurfacing and smooth shading in volume rendering [S2][S8, unofficial]. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md), which produces a piecewise-constant field with sharp discontinuities, this method blends adjacent voxels to ensure smoothness [S1, unofficial][S2]. It differs from [B-spline Field Interpolation](11066-b-spline-field-interpolation.md) by using a fixed 4x4x4 support region defined by the grid topology rather than a general convolution kernel, and from [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) by operating in 3D with cubic basis functions [S1, unofficial][S2].

## How it works
**Inputs:**
- A query point $p = (p_x, p_y, p_z)$ in world space.
- A dense grid structure `grid` defining the spatial domain, bounding box, and voxel dimensions [S2].
- A discrete scalar field `data` storing values at grid vertices [S1, unofficial].

**Outputs:**
- A scalar value $V(p)$ representing the interpolated density or intensity at $p$ [S1, unofficial].
- A gradient vector $\nabla V(p) = (\frac{\partial V}{\partial x}, \frac{\partial V}{\partial y}, \frac{\partial V}{\partial z})$ representing the field's derivative at $p$ [S2].

**Data Structures:**
- `Grid`: Contains bounding box bounds, voxel dimensions (`voxel_dim`), and dimensions in voxels (`xdim`, `ydim`, `zdim`) [S2].
- `Data`: A 3D array or tensor indexed by integer coordinates $(i, j, k)$ corresponding to grid vertices [S1, unofficial].

**Algorithm Steps:**

1.  **Coordinate Mapping:**
    Compute the integer voxel coordinates $v_i$ containing the query point $p$:
    $$ v_i[\text{axis}] = \lfloor \frac{p[\text{axis}] - \text{bounds}[0][\text{axis}]}{\text{voxel\_dim}[\text{axis}]} \rfloor $$
    where `bounds[0]` is the minimum corner of the grid [S1, unofficial].

2.  **Local Offset Calculation:**
    Compute the minimum corner $v_{\min}$ of the voxel containing $p$:
    $$ v_{\min} = \text{bounds}[0] + v_i \times \text{voxel\_dim} $$
    Compute the fractional offset $t$ inside the voxel, normalized to $[0, 1]$:
    $$ t = \frac{p - v_{\min}}{\text{voxel\_dim}} $$
    [S1, unofficial]

3.  **Neighborhood Selection:**
    Identify the 4x4x4 neighborhood of voxels surrounding $v_i$. The indices range from $v_i[\text{axis}] - 1$ to $v_i[\text{axis}] + 2$ for each axis [S1, unofficial].

4.  **1D Cubic Interpolation (Hermite Basis):**
    The method uses a cubic Hermite basis function `CINT_CR` (or equivalent) to interpolate along one axis. For an interval $[x_k, x_{k+1}]$ with values $p_k, p_{k+1}$ and derivatives (or scaled differences) $m_k, m_{k+1}$, the interpolation at parameter $t \in [0, 1]$ is:
    $$
    \begin{aligned}
    \Delta &= x_{k+1} - x_k \\
    t &= \max\left(\frac{x - x_k}{\Delta}, 0\right) \\
    t_2 &= t^2, \quad t_3 = t^3 \\
    h_{00} &= 2t_3 - 3t_2 + 1 \\
    h_{10} &= t_3 - 2t_2 + t \\
    h_{01} &= -2t_3 + 3t_2 \\
    h_{11} &= t_3 - t_2 \\
    \text{Result} &= h_{00} p_k + h_{10} \Delta m_k + h_{01} p_{k+1} + h_{11} \Delta m_{k+1}
    \end{aligned}
    $$
    [S1, unofficial]
    *Note: In standard tri-cubic interpolation of scalar fields without stored derivatives, $m_k$ and $m_{k+1}$ are often approximated using finite differences of neighboring scalar values, or the method assumes the "CINT_CR" function handles the specific derivative estimation required by the implementation [S1, unofficial][S2].*

5.  **Tri-Cubic Evaluation:**
    Perform interpolation sequentially along the Z, Y, and X axes:
    a.  **Z-Interpolation:** For each $j \in [-1, 2]$ and $i \in [-1, 2]$, compute 1D cubic interpolation along the Z-axis using 4 values:
        $$ z\_interp[j+1][i+1] = \text{CINT\_CR}(t_z, \text{data}[v_i+x+i][v_i+y+j][v_i+z-1], \dots, \text{data}[v_i+x+i][v_i+y+j][v_i+z+2]) $$
        [S1, unofficial]
    b.  **Y-Interpolation:** For each $i \in [0, 3]$, interpolate the results from step (a) along the Y-axis:
        $$ y\_interp[i] = \text{CINT\_CR}(t_y, z\_interp[0][i], z\_interp[1][i], z\_interp[2][i], z\_interp[3][i]) $$
        [S1, unofficial]
    c.  **X-Interpolation:** Interpolate the results from step (b) along the X-axis to get the final scalar value:
        $$ V(p) = \text{CINT\_CR}(t_x, y\_interp[0], y\_interp[1], y\_interp[2], y\_interp[3]) $$
        [S1, unofficial]

6.  **Gradient Computation:**
    The gradient $\nabla V(p)$ can be computed analytically if the basis functions' derivatives are known, or by computing discrete gradients at vertices and interpolating them similarly [S2].
    -   **Analytical:** Differentiate the basis functions $h_{00}, h_{10}, h_{01}, h_{11}$ with respect to $t$ and scale by $1/\text{voxel\_dim}$ for each axis [S2].
    -   **Discrete:** Use `gradients.compute_discrete_gradients(values)` to precompute vertex gradients, then interpolate these gradient vectors using the same tri-cubic scheme [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tri_cubic_interpolate(p, grid, data):
    """
    Computes scalar value and gradient at point p using tri-cubic interpolation.
    """
    # 1. Coordinate Mapping
    v_i = [0, 0, 0]
    v_min = [0.0, 0.0, 0.0]
    t = [0.0, 0.0, 0.0]
    
    for axis in range(3):
        # Compute integer index
        v_i[axis] = floor((p[axis] - grid.bounds[0][axis]) / grid.voxel_dim[axis])
        
        # Compute min corner and fractional offset
        v_min[axis] = grid.bounds[0][axis] + v_i[axis] * grid.voxel_dim[axis]
        t[axis] = (p[axis] - v_min[axis]) / grid.voxel_dim[axis]
        
        # Clamp t to [0, 1] to handle boundary cases precisely
        t[axis] = max(0.0, min(1.0, t[axis]))

    # 2. Helper: 1D Cubic Hermite Interpolation
    # Assumes derivatives m_k, m_{k+1} are passed or computed via finite differences
    def cint_cr(t_val, p0, m0, p1, m1):
        t2 = t_val * t_val
        t3 = t2 * t_val
        h00 = 2 * t3 - 3 * t2 + 1
        h10 = t3 - 2 * t2 + t_val
        h01 = -2 * t3 + 3 * t2
        h11 = t3 - t2
        return h00 * p0 + h10 * m0 + h01 * p1 + h11 * m1

    # 3. Tri-Cubic Evaluation
    # We need 4x4x4 values. Indices relative to v_i are -1, 0, 1, 2.
    # To simplify, we assume data access handles boundary checks or padding.
    
    # Precompute Z-interpolations for all x, y slices
    # z_interp[y_idx][x_idx]
    z_interp = [[0.0 for _ in range(4)] for _ in range(4)]
    
    for j in range(-1, 3): # y offset
        for i in range(-1, 3): # x offset
            # Get 4 z-values: v_i[z]-1 to v_i[z]+2
            idx_z0 = v_i[2] - 1
            idx_z1 = v_i[2]
            idx_z2 = v_i[2] + 1
            idx_z3 = v_i[2] + 2
            
            # Access data[i][j][z]
            # Note: In a real implementation, bounds checking is required.
            val0 = data[v_i[0] + i][v_i[1] + j][idx_z0]
            val1 = data[v_i[0] + i][v_i[1] + j][idx_z1]
            val2 = data[v_i[0] + i][v_i[1] + j][idx_z2]
            val3 = data[v_i[0] + i][v_i[1] + j][idx_z3]
            
            # Approximate derivatives for z-axis using central differences if not stored
            # m_z0 = (val1 - val0) / grid.voxel_dim[2]
            # m_z1 = (val2 - val1) / grid.voxel_dim[2]
            # m_z2 = (val3 - val2) / grid.voxel_dim[2]
            # However, standard tri-cubic often uses Catmull-Rom or similar which implies specific derivative handling.
            # Using the formula from [S1, unofficial] which calls CINT_CR with 4 values implies a specific basis.
            # If CINT_CR takes 4 values, it might be Catmull-Rom. 
            # The formula in [S1, unofficial] shows h00, h10 etc which is Hermite. 
            # We assume m0, m1 are derivatives. If only values are available, we must estimate them.
            # For this pseudocode, we assume a helper `get_derivatives` exists or we use finite diffs.
            
            # Simplified: Using Catmull-Rom logic for 4 points (common in "tri-cubic" without explicit gradients)
            # CR(t, p0, p1, p2, p3) = 0.5 * ((2*p1) + (-p0+p2)*t + (2*p0-5*p1+4*p2-p3)*t^2 + (-p0+3*p1-3*p2+p3)*t^3)
            # But [S1, unofficial] explicitly lists Hermite basis functions. We stick to Hermite with estimated derivatives.
            
            dz = grid.voxel_dim[2]
            m0 = (val1 - val0) / dz
            m1 = (val2 - val1) / dz
            
            z_interp[j+1][i+1] = cint_cr(t[2], val1, m0, val2, m1)

    # Y-Interpolation
    y_interp = [0.0 for _ in range(4)]
    for i in range(4):
        # z_interp indices 0..3 correspond to y offsets -1..2
        val0 = z_interp[0][i]
        val1 = z_interp[1][i]
        val2 = z_interp[2][i]
        val3 = z_interp[3][i]
        
        dy = grid.voxel_dim[1]
        m0 = (val1 - val0) / dy
        m1 = (val2 - val1) / dy
        
        y_interp[i] = cint_cr(t[1], val1, m0, val2, m1)

    # X-Interpolation
    val0 = y_interp[0]
    val1 = y_interp[1]
    val2 = y_interp[2]
    val3 = y_interp[3]
    
    dx = grid.voxel_dim[0]
    m0 = (val1 - val0) / dx
    m1 = (val2 - val1) / dx
    
    result = cint_cr(t[0], val1, m0, val2, m1)
    
    return result
```

**Parameters:**
- `grid`: The spatial indexing structure, typically `CGAL::Isosurfacing::Cartesian_grid_3` [S2].
- `voxel_dim`: The physical size of each voxel [S1, unofficial].
- `isovalue`: Used in downstream applications like Marching Cubes or Dual Contouring, but not in the interpolation itself [S2].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** If the query point $p$ lies outside the grid bounds, the index calculation $v_i$ may produce out-of-bounds array accesses. A guard must clamp $v_i$ or return a default background value [S1, unofficial][S2].
- **Derivative Estimation:** If the discrete field does not store explicit gradients, derivatives must be estimated (e.g., via finite differences). This can introduce noise or inaccuracies compared to analytically derived gradients [S2].
- **Precision:** Using `float` for high-resolution grids may lead to precision loss in offset calculation $t$. Double precision is recommended for scientific applications [S2].

## Subtopics

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Extracts the 8 vertices of the enclosing grid cell. It is the foundational data retrieval step for trilinear interpolation, solving the mapping of continuous coordinates to discrete tensor indices.

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
Retrieves the single scalar value of the nearest voxel without interpolation. It preserves sharp discontinuities but introduces aliasing, unlike the smooth blending of this method.

### [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md)
Uses two piecewise quadratic Bézier curves to satisfy Hermite data (value and derivative) at endpoints. It provides $C^1$ continuity in 1D with better stability for small subdivisions than cubic interpolation.

### [Mipmap Grid Query](4154-mipmap-grid-query.md)
Selects grid resolution based on spatial distance from the center. It solves adaptive resolution sampling, whereas this method uses a fixed resolution grid.

### [Cylindrical Coordinate Interpolation](4282-cylindrical-coordinate-interpolation.md)
Maps Cartesian points to cylindrical space before sampling. It reduces artifacts for radially symmetric data by aligning the coordinate system with the geometry's principal axis.

### [B-spline Field Interpolation](11066-b-spline-field-interpolation.md)
Evaluates a continuous function using cubic B-splines over a 4x4x4 support grid. It provides $C^2$ continuity and is distinguished by its use of polynomial basis functions that sum to 1, offering smoother derivatives than the standard tri-cubic approach.

### Dense Copy Extraction
Extracts a region defined by a bounding box into a dense 3D array, filled with the grid's background value, then overwritten with `get_value` for every coordinate in the box [S3, unofficial]. It calculates dimensions $dx, dy, dz$ from the bbox, initializes the result array, and uses a triple nested loop over $ix, iy, iz$ to compute coordinates and set `result[ix + 1, iy + 1, iz + 1] = get_value(tree, c)` [S3, unofficial].

### Uncertain Gradient Field Interpolation
Reconstructs the gradient field from an uncertain scalar field, resulting in a trivariate random vector $Y$ representing uncertainties of partial derivatives [S4]. The gradient is computed as a linear combination of random variables $X_i$ at neighboring voxels: $\begin{bmatrix} Y_x \\ Y_y \\ Y_z \end{bmatrix} = \sum_i \begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} X_i$, where weights are determined by the basis function $\rho(\mathbf{v} - \mathbf{v}_i)$ [S4]. Voxels outside the support of $\rho$ are assigned zero weights [S4].

### Backprojection via Shepard's Interpolation
Fits a continuous scale field $S(x)$ from a discrete set of LoG extrema $N=\{(x_i, t_i)\}$ using basis functions [S5, unofficial]. The scale field is $S(x) = \sum_{x_i \in N} \Theta(||x - x_i||) t_i$, where $\Theta(d)$ is a fourth-degree Wendland polynomial with compact support [S5, unofficial]. Alternatively, the maximum operator $S(x) = \max_{x_i \in N} \{\Theta(||x - x_i||) t_i\}$ can be used to prioritize larger features [S5, unofficial]. A parameter $h$ controls the local support, typically $h = k \cdot t_i$ [S5, unofficial].

### Kriging Interpolation
Constructs a regular grid data model from limited discrete 3D borehole data [S6]. It involves gridding, selecting reference points $p_i$, calculating coefficients $K, \lambda, D$ based on a variation function, and solving the equation set using the LU method to get weight coefficients $\lambda_i$ [S6]. The estimated value is $R'(p_0) = \sum_{i=1}^n \lambda_i R(p_i)$ [S6].

### Data Interpolation
Preprocesses irregular or non-uniform grid data into regular grids for volume texture generation [S7, unofficial]. For ocean environmental parameters, spline interpolation is applied vertically (0 m to -2000 m, 5 m interval, 401 layers) [S7, unofficial]. For acoustic propagation loss, cubic spline interpolation is applied horizontally to generate 300x300 regular latitude/longitude grids per depth layer [S7, unofficial].

### Hermite Interpolation of Scalar Fields
Renders scalar valued functions (height and signed distance fields) by storing partial derivatives with function values and using Hermite interpolation to guarantee globally $C^1$-continuous results [S8, unofficial]. It uses exact normals of the reconstructed cubic Hermite surface as shading normals and derives approximations for mixed partial derivatives for 3D first-order fields [S8, unofficial]. It serves as a storage-efficient replacement for normal mapping [S8, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, as it involves a fixed number of arithmetic operations (64 value accesses and polynomial evaluations) regardless of grid size [S1, unofficial].
- **Memory Complexity:** $O(N)$ where $N$ is the number of voxels in the grid, plus $O(N)$ if gradients are precomputed and stored [S2].
- **Performance:** The method is computationally intensive per sample compared to trilinear interpolation due to the 64-neighbor access and cubic basis calculations, but it provides significantly higher visual quality for surface extraction [S1, unofficial][S2].
- **Strengths:** Provides smooth, continuous derivatives ($C^1$) which are critical for accurate normal estimation in isosurfacing (e.g., Dual Contouring) and smooth shading [S2][S8, unofficial].
- **Limitations:** Requires more memory than nearest-neighbor or trilinear methods if gradients are stored. Boundary handling requires careful clamping or padding [S1, unofficial][S2].
- **When to Use:** When smooth surface reconstruction or accurate gradient-based shading is required, such as in medical imaging isosurfacing or high-quality volume rendering [S2][S8, unofficial].
- **When Not to Use:** For real-time applications with strict memory constraints where visual smoothness is secondary to performance, or for data with sharp discontinuities where piecewise-constant interpolation is preferred [S1, unofficial][S2].

## Background
The method originates from the need to interpolate discrete scalar fields for isosurfacing and volume rendering. It is implemented in libraries such as CGAL's Isosurfacing module (`CGAL::Isosurfacing::Interpolated_discrete_values_3` and `CGAL::Isosurfacing::Interpolated_discrete_gradients_3`) for contouring discrete data [S2]. The underlying cubic spline interpolation techniques are well-established in computer graphics for smooth field evaluation [S1, unofficial]. Variants like Hermite interpolation for scalar fields have been applied to accelerate sphere tracing and signed distance field rendering [S8, unofficial]. Uncertain gradient interpolation has been developed for nonparametric models of uncertainty in volume rendering [S4].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S3] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S4] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
[S5] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
[S6] Volume Rendering of 3D Borehole Data Based on GPU, https://doi.org/10.2991/icsem.2013.181, fetched 2026-10-26
[S7] The Implementation of a WebGPU-Based Volume Rendering Framework ... - MDPI, https://www.mdpi.com/2076-3417/15/5/2782, fetched 2026-10-29 (unofficial)
[S8] (PDF) Accelerating Sphere Tracing - ResearchGate, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
