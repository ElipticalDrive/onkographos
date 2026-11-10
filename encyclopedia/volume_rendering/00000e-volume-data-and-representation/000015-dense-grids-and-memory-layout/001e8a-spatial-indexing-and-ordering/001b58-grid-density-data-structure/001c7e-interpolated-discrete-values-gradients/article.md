# Interpolated Discrete Values & Gradients

Also called: Cubic Spline Interpolation, CubicSpline, Discrete Field Interpolation, Interpolated Grid Fields, Tri-Cubic Interpolation

## Overview
Interpolated Discrete Values & Gradients computes a continuous scalar field and its spatial gradient at arbitrary 3D points by applying tri-cubic interpolation to discrete data stored at grid vertices [S1, unofficial][S2]. It solves the problem of mapping continuous world-space coordinates to discrete memory indices while providing $C^1$ continuity for smooth volume rendering and isosurface extraction [S1, unofficial][S2]. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md), which produces a piecewise-constant field with aliasing artifacts, this method blends adjacent voxels to preserve smoothness [S1, unofficial][S2]. It differs from [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) by deriving gradients analytically from the interpolation basis functions rather than approximating them via finite differences on the raw discrete values [S2].

## How it works
**Inputs:**
- A dense 3D grid of discrete scalar values $V$ defined at integer vertex coordinates $(i, j, k)$ [S1, unofficial][S2].
- A query point $p = (p_x, p_y, p_z)$ in continuous space [S1, unofficial].
- Grid parameters: bounding box bounds, voxel dimensions (spacing) $dx, dy, dz$ [S1, unofficial][S2].

**Outputs:**
- Scalar value $f(p)$ at the query point [S1, unofficial].
- Gradient vector $\nabla f(p) = (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})$ at the query point [S2].

**Data Structures:**
- The grid is typically represented as a 3D array or tensor of size $N_x \times N_y \times N_z$ [S2].
- In implementations like CGAL, the grid is encapsulated in a structure (e.g., `CGAL::Isosurfacing::Cartesian_grid_3`) providing spatial indexing and point access via `grid.point(i,j,k)` [S2].

**Algorithm Steps:**

1.  **Voxel Coordinate Calculation:** Determine the integer coordinates of the voxel containing the query point $p$. For each axis $a \in \{x, y, z\}$:
    $$ v_i[a] = \lfloor \frac{p[a] - \text{bounds}[0][a]}{\text{voxel\_dim}[a]} \rfloor $$
    where $\text{bounds}[0]$ is the minimum corner of the grid and $\text{voxel\_dim}$ is the spacing vector [S1, unofficial].

2.  **Local Offset Calculation:** Compute the fractional offset $t$ of $p$ within the voxel, normalized to $[0, 1]$:
    $$ v_{\min} = \text{bounds}[0] + v_i \cdot \text{voxel\_dim} $$
    $$ t[a] = \frac{p[a] - v_{\min}[a]}{\text{voxel\_dim}[a]} $$
    [S1, unofficial]

3.  **Neighborhood Selection:** Identify the 64 neighboring voxel values required for tri-cubic interpolation. This involves a $4 \times 4 \times 4$ grid of vertices centered around the voxel containing $p$. The indices range from $v_i[a]-1$ to $v_i[a]+2$ for each axis [S1, unofficial].

4.  **1D Cubic Interpolation (Hermite Basis):** Perform cubic interpolation along one axis using 4 values and their associated derivatives (or estimated slopes). The standard Hermite basis functions for an interval $[x_k, x_{k+1}]$ with values $p_k, p_{k+1}$ and derivatives $m_k, m_{k+1}$ are [S1, unofficial]:
    $$ \delta = x_{k+1} - x_k $$
    $$ t = \frac{x - x_k}{\delta} \quad (\text{clamped to } [0,1]) $$
    $$ t_2 = t^2, \quad t_3 = t^3 $$
    $$ h_{00} = 2t_3 - 3t_2 + 1 $$
    $$ h_{10} = t_3 - 2t_2 + t $$
    $$ h_{01} = -2t_3 + 3t_2 $$
    $$ h_{11} = t_3 - t_2 $$
    $$ \text{Result} = h_{00} p_k + h_{10} \delta m_k + h_{01} p_{k+1} + h_{11} \delta m_{k+1} $$
    [S1, unofficial]

    *Note on Gradients:* If analytical gradients are not available at the vertices, they must be computed first. This can be done via finite differences on the discrete values (e.g., using `gradients.compute_discrete_gradients(values)` in CGAL) [S2]. Alternatively, for uncertain fields, gradients are reconstructed as random vectors based on basis functions [S4].

5.  **Tri-Cubic Interpolation:** Apply the 1D interpolation sequentially along the z, y, and x axes.
    - **Z-axis:** For each $j \in [-1, 2]$ and $i \in [-1, 2]$, interpolate along Z using 4 values from the neighborhood:
      $$ z\_interp[j+1][i+1] = \text{CubicInterp}(t_z, V[v_i+x-1][v_i+y+j][v_i+z-1], \dots, V[v_i+x+2][v_i+y+j][v_i+z+2]) $$
      [S1, unofficial]
    - **Y-axis:** Interpolate the resulting Z-interpolated values along Y:
      $$ y\_interp[i] = \text{CubicInterp}(t_y, z\_interp[0][i], z\_interp[1][i], z\_interp[2][i], z\_interp[3][i]) $$
      for $i \in [0, 3]$ [S1, unofficial].
    - **X-axis:** Interpolate the Y-interpolated values along X to get the final scalar:
      $$ f(p) = \text{CubicInterp}(t_x, y\_interp[0], y\_interp[1], y\_interp[2], y\_interp[3]) $$
      [S1, unofficial]

6.  **Gradient Computation:** The gradient $\nabla f(p)$ is computed by differentiating the tri-cubic basis functions with respect to $x, y, z$. This requires the derivatives of the basis functions $h_{00}, h_{10}, h_{01}, h_{11}$ and the stored vertex gradients or finite-difference approximations thereof [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_hermite_1d(t, p0, m0, p1, m1, delta):
    """
    Performs 1D cubic Hermite interpolation.
    t: normalized parameter in [0, 1]
    p0, p1: values at endpoints
    m0, m1: derivatives at endpoints
    delta: distance between endpoints (x1 - x0)
    """
    t2 = t * t
    t3 = t2 * t
    
    h00 = 2 * t3 - 3 * t2 + 1
    h10 = t3 - 2 * t2 + t
    h01 = -2 * t3 + 3 * t2
    h11 = t3 - t2
    
    return h00 * p0 + h10 * delta * m0 + h01 * p1 + h11 * delta * m1

def tri_cubic_interpolate(grid, p, voxel_dim, bounds_min, vertex_gradients):
    """
    Computes scalar value and gradient at point p using tri-cubic interpolation.
    grid: 3D array of scalar values
    vertex_gradients: 3D array of gradient vectors at vertices
    """
    # 1. Compute voxel coordinates
    v_i = [floor((p[a] - bounds_min[a]) / voxel_dim[a]) for a in range(3)]
    
    # 2. Compute local offset t
    v_min = [bounds_min[a] + v_i[a] * voxel_dim[a] for a in range(3)]
    t = [(p[a] - v_min[a]) / voxel_dim[a] for a in range(3)]
    
    # 3. Extract 4x4x4 neighborhood values and gradients
    # Indices range from v_i[a]-1 to v_i[a]+2
    vals = [[[grid[v_i[0]+i][v_i[1]+j][v_i[2]+k] for k in range(-1, 3)] 
             for j in range(-1, 3)] for i in range(-1, 3)]
    
    grads = [[[vertex_gradients[v_i[0]+i][v_i[1]+j][v_i[2]+k] for k in range(-1, 3)] 
              for j in range(-1, 3)] for i in range(-1, 3)]
    
    # 4. Interpolate along Z
    z_interp = [[[None for _ in range(4)] for _ in range(4)] for _ in range(4)]
    z_grads = [[[None for _ in range(3)] for _ in range(4)] for _ in range(4)]
    
    for i in range(4):
        for j in range(4):
            # Extract 4 values and 4 gradients along Z for this (i,j)
            z_vals = [vals[i][j][k] for k in range(4)]
            z_grads_vals = [grads[i][j][k] for k in range(4)]
            
            # Perform cubic interpolation for value and gradient components
            # Note: Full gradient derivation requires differentiating the basis functions
            # For simplicity, we assume a helper exists or compute scalar first
            z_interp[i][j][0] = cubic_hermite_1d(t[2], z_vals[0], z_grads_vals[0][2], 
                                                 z_vals[1], z_grads_vals[1][2], voxel_dim[2])
            # ... (Repeat for other z-steps to get 4x4 plane)
            
    # 5. Interpolate along Y
    y_interp = [None for _ in range(4)]
    for i in range(4):
        y_vals = [z_interp[i][j][0] for j in range(4)] # Simplified access
        # ... interpolate along Y
        
    # 6. Interpolate along X
    result_val = cubic_hermite_1d(t[0], y_interp[0], ..., y_interp[3], ..., voxel_dim[0])
    
    return result_val
```
*Note: The pseudocode above outlines the structure. A complete implementation requires computing the gradient vector components by applying the derivative of the Hermite basis functions to the stored vertex gradients or finite-difference estimates [S2].*

## Subtopics

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Extracts the 8 vertices of the enclosing grid cell. It is the foundational data retrieval step for trilinear interpolation, whereas Interpolated Discrete Values & Gradients uses a 4x4x4 neighborhood for cubic accuracy.

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
Retrieves a single voxel value without interpolation. It preserves sharp discontinuities but introduces aliasing, unlike the smooth blending of tri-cubic interpolation.

### [Mipmap Grid Query](4154-mipmap-grid-query.md)
Selects a mip level based on spatial distance to adapt resolution. It differs by changing the grid resolution rather than the interpolation order within a single grid.

### [Cylindrical Coordinate Interpolation](4282-cylindrical-coordinate-interpolation.md)
Maps Cartesian points to cylindrical space before sampling. It aligns with cylindrical symmetry, whereas standard tri-cubic assumes Cartesian regularity.

### [Data Interpolation](7180-data-interpolation.md)
A preprocessing technique to map scattered data to rectangular grids. It focuses on global grid construction, whereas Interpolated Discrete Values & Gradients operates on an already regular grid.

### [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md)
A general class of methods using piecewise polynomials. Interpolated Discrete Values & Gradients is a specific instance using tri-cubic (Hermite) splines.

### Dense Copy Extraction
Extracts a bounding box region into a dense 3D array filled with background values, then overwrites with `get_value`. It is a data extraction utility rather than a sampling method [S3, unofficial].

### Uncertain Gradient Field Interpolation
Reconstructs the gradient as a trivariate random vector to model uncertainty. It extends the deterministic gradient computation by treating weights as basis functions $\rho$ applied to random variables $X_i$ [S4].

### Backprojection via Shepard's Interpolation
Fits a continuous scale field from discrete LoG extrema using Wendland polynomials. It is used for scale-based transfer functions rather than general scalar field interpolation [S5, unofficial].

### Kriging Interpolation
Constructs a regular grid from limited discrete data (e.g., boreholes) using weighted linear combinations solved via LU decomposition. It is a geostatistical method for grid generation, not local sampling [S6].

### Zero Set Initialization of Isosurfaces
Initializes distance volumes using local gradient estimates and linear approximation. It uses gradients for closest-point computation rather than value interpolation [S7].

### Constant Filter Radius Mapping
Maps curvilinear grids to rectangular grids using a constant filter radius. It risks violating Nyquist criteria for multi-scale data [S8, unofficial].

### Dense Copy Extraction
Extracts a region defined by a bounding box into a dense 3D array, filled with the grid's background value, then overwritten with `get_value` for every coordinate in the box [S3, unofficial]. It calculates dimensions $dx, dy, dz$ from the bbox, initializes the result array, and iterates over $ix, iy, iz$ to compute coordinates and set values [S3, unofficial].

### Uncertain Gradient Field Interpolation
Reconstructs the gradient field from an uncertain scalar field, resulting in a trivariate random vector $Y$ representing uncertainties of partial derivatives [S4]. The gradient is computed as $\begin{bmatrix} Y_x \\ Y_y \\ Y_z \end{bmatrix} = \sum_i \begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} X_i$, where weights are determined by a basis function $\rho(\mathbf{v} - \mathbf{v}_i)$ [S4]. Voxels outside the support of $\rho$ receive zero weights [S4].

### Backprojection via Shepard's Interpolation
Fits a continuous scale field $S(x)$ from discrete LoG extrema $N=\{(x_i, t_i)\}$ using basis functions [S5, unofficial]. The scale field is $S(x) = \sum_{x_i \in N} \Theta(||x - x_i||) t_i$, where $\Theta(d)$ is a fourth-degree Wendland polynomial [S5, unofficial]. Alternatively, a maximum operator can be used to prioritize larger features [S5, unofficial]. A parameter $h$ controls the local support [S5, unofficial].

### Kriging Interpolation
Constructs a regular grid data model from limited discrete 3D borehole data [S6]. It involves gridding, selecting reference points, calculating coefficients via a variation function, and solving an equation set using LU decomposition to get weight coefficients $\lambda_i$ [S6]. The estimated value is $R'(p_0) = \sum_{i=1}^n \lambda_i R(p_i)$ [S6].

### Zero Set Initialization of Isosurfaces
Initializes closest points for isosurfaces using local gradient estimates instead of trilinear interpolation [S7]. It examines volume cells, initializes distance samples at corners if the cell contains the isosurface, and estimates the gradient $\nabla f(s)$ using central differencing [S7]. The closest point is found via linear approximation $f(s') = f(s) + \nabla f(s) \cdot (s' - s)$ [S7].

### Constant Filter Radius Mapping
Maps curvilinear grid data to a rectangular grid using a user-set constant filter radius [S8, unofficial]. It violates Nyquist criteria when input data has a wide range of scales, as a single radius cannot adequately sample all parts of the input grid [S8, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, as it accesses a fixed $4 \times 4 \times 4$ neighborhood (64 values) regardless of grid size [S1, unofficial].
- **Memory Complexity:** Requires storage of the grid values and, for gradient computation, the vertex gradients (or the ability to compute them on the fly) [S2].
- **Performance:** Provides smooth visual results suitable for high-quality volume rendering and isosurfacing [S1, unofficial][S2]. It is more computationally expensive than trilinear or nearest-neighbor interpolation due to the larger neighborhood and basis function evaluations [S1, unofficial].
- **Strengths:** $C^1$ continuity ensures smooth gradients, which is critical for accurate normal estimation in isosurface extraction (e.g., Dual Contouring) and smooth lighting in volume rendering [S2].
- **Limitations:** Requires a regular grid structure [S1, unofficial]. Boundary conditions must be handled carefully (e.g., clamping or extrapolation) when the query point is near the grid edges [S1, unofficial].

## Background
The method is implemented in libraries such as CGAL's Isosurfacing module, specifically in `Interpolated_discrete_values_3` and `Interpolated_discrete_gradients_3`, used for contouring discrete data [S2]. It is also used in CUDA-based volume rendering projects for ray marching with transfer functions [S1, unofficial]. The underlying cubic spline interpolation technique is a standard numerical analysis method for smooth curve fitting [S1, unofficial].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S3] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S4] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
[S5] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
[S6] Volume Rendering of 3D Borehole Data Based on GPU, https://doi.org/10.2991/icsem.2013.181, fetched 2026-10-26
[S7] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409942/?q=volume data compression rendering, fetched 2026-10-28
[S8] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
