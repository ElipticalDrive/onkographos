# Interpolated Discrete Values & Gradients

Also called: Cubic Spline Interpolation, CubicSpline, Discrete Field Interpolation, Interpolated Grid Fields

## Overview
Interpolated Discrete Values & Gradients computes a continuous scalar field and its spatial gradient from discrete grid data using tri-cubic interpolation [S1, unofficial][S2]. It solves the problem of mapping continuous world-space coordinates to discrete memory indices while providing smooth, differentiable field values essential for ray marching and isosurface extraction [S1, unofficial][S2]. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md), which produces a piecewise-constant field with sharp discontinuities, this method blends adjacent voxels to ensure $C^0$ continuity and provides explicit gradient estimates for stable normal estimation [S1, unofficial][S2]. It differs from [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) by deriving gradients analytically from the interpolation basis functions rather than approximating them via discrete differences, resulting in smoother derivatives [S2].

## How it works
**Inputs:**
- A query point $p = (p_x, p_y, p_z)$ in continuous space.
- A dense 3D grid of scalar values $V$ with dimensions $D_x \times D_y \times D_z$.
- Grid bounds $B_{min}$ and voxel dimensions $d_x, d_y, d_z$.

**Outputs:**
- Scalar value $f(p)$.
- Gradient vector $\nabla f(p) = (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})$.

**Data Structures:**
- The grid $V$ is stored as a contiguous 3D array or flattened 1D array. Accessor `field(data, i, j, k)` returns the value at integer indices $(i, j, k)$.
- The interpolation uses a $4 \times 4 \times 4$ neighborhood of 64 voxels centered around the query point's voxel.

**Steps:**

1.  **Voxel Coordinate Calculation:**
    Determine the integer coordinates of the voxel containing $p$:
    $$ v_i[axis] = \lfloor \frac{p[axis] - B_{min}[axis]}{d[axis]} \rfloor $$
    where $axis \in \{x, y, z\}$ [S1, unofficial].

2.  **Local Offset Calculation:**
    Compute the minimum corner of the voxel containing $p$:
    $$ v_{min}[axis] = B_{min}[axis] + v_i[axis] \cdot d[axis] $$
    Compute the fractional offset $t$ in range $[0, 1]$:
    $$ t[axis] = \frac{p[axis] - v_{min}[axis]}{d[axis]} $$
    [S1, unofficial].

3.  **1D Cubic Interpolation Basis:**
    The method uses cubic Hermite spline interpolation (often referred to as tri-cubic in 3D) along each axis. For an interval $[x_k, x_{k+1}]$ with values $p_k, p_{k+1}$ and derivatives $m_k, m_{k+1}$, the interpolation function $CINT\_CR(t, p_k, m_k, p_{k+1}, m_{k+1})$ is defined as [S1, unofficial]:
    $$ \Delta = x_{k+1} - x_k $$
    $$ t = \frac{\max(x - x_k, 0)}{\Delta} $$
    $$ t_2 = t^2, \quad t_3 = t^3 $$
    $$ h_{00} = 2t^3 - 3t^2 + 1 $$
    $$ h_{10} = t^3 - 2t^2 + t $$
    $$ h_{01} = -2t^3 + 3t^2 $$
    $$ h_{11} = t^3 - t^2 $$
    $$ Result = h_{00} p_k + h_{10} \Delta m_k + h_{01} p_{k+1} + h_{11} \Delta m_{k+1} $$
    [S1, unofficial].

    *Note on Gradients:* The derivatives $m_k, m_{k+1}$ required for the Hermite basis are typically computed using finite differences or provided by a separate gradient field. If not provided, `gradients.compute_discrete_gradients(values)` can be used to precompute them [S2].

4.  **Tri-Cubic Interpolation (Value):**
    Perform interpolation along the $z$-axis first, then $y$, then $x$.
    
    For each $j \in [-1, 2]$ and $i \in [-1, 2]$:
    $$ z\_interp[j+1][i+1] = CINT\_CR(t_z, V[v_i+x+i][v_i+y+j][v_i+z-1], m_z[v_i+x+i][v_i+y+j][v_i+z-1], V[v_i+x+i][v_i+y+j][v_i+z], m_z[v_i+x+i][v_i+y+j][v_i+z], V[v_i+x+i][v_i+y+j][v_i+z+1], m_z[v_i+x+i][v_i+y+j][v_i+z+1], V[v_i+x+i][v_i+y+j][v_i+z+2], m_z[v_i+x+i][v_i+y+j][v_i+z+2]) $$
    [S1, unofficial].
    
    *Correction:* The standard tri-cubic implementation described in [S1, unofficial] simplifies the gradient dependency by assuming the derivatives are handled within the `CINT_CR` call or by using a basis that only requires values (like Catmull-Rom). However, the explicit formula in [S1, unofficial] lists derivatives $m_k$. If only values are available, Catmull-Rom splines are a common variant where $m_k = \frac{1}{2}(V_{k+1} - V_{k-1})$. The source [S1, unofficial] explicitly lists `CINT_CR` taking 4 arguments in the loop description but 8 in the formula. The loop description `CINT_CR(t.z, field(..., -1), field(..., 0), field(..., 1), field(..., 2))` suggests a 4-point stencil, characteristic of Catmull-Rom or B-spline, not Hermite which requires derivatives. Given the alias "Cubic Spline Interpolation" and the 4-argument call in the loop, the implementation likely uses a value-only cubic basis (like Catmull-Rom) where derivatives are implicit. We will assume the 4-argument form for implementation clarity, mapping to Catmull-Rom coefficients if derivatives are not explicitly stored [S1, unofficial].
    
    Re-evaluating [S1, unofficial] loop: `CINT_CR(t.z, field(data, v_i[0]+i, v_i[1]+j, v_i[2]-1), field(data, v_i[0]+i, v_i[1]+j, v_i[2]), field(data, v_i[0]+i, v_i[1]+j, v_i[2]+1), field(data, v_i[0]+i, v_i[1]+j, v_i[2]+2))`. This takes 4 values. This corresponds to Catmull-Rom interpolation.
    
    Catmull-Rom Basis for $t \in [0,1]$:
    $$ h_0 = -0.5t^3 + t^2 - 0.5t $$
    $$ h_1 = 1.5t^3 - 2.5t^2 + 1 $$
    $$ h_2 = -1.5t^3 + 2t^2 + 0.5t $$
    $$ h_3 = 0.5t^3 - 0.5t^2 $$
    $$ Result = h_0 P_{-1} + h_1 P_0 + h_2 P_1 + h_3 P_2 $$
    
    Interpolate along $y$:
    $$ y\_interp[i] = CINT\_CR(t_y, z\_interp[0][i], z\_interp[1][i], z\_interp[2][i], z\_interp[3][i]) \quad \text{for } i \in [0, 3] $$
    [S1, unofficial].
    
    Interpolate along $x$:
    $$ result = CINT\_CR(t_x, y\_interp[0], y\_interp[1], y\_interp[2], y\_interp[3]) $$
    [S1, unofficial].

5.  **Gradient Calculation:**
    The gradient $\nabla f(p)$ is computed by differentiating the interpolation basis functions. If using Catmull-Rom, the gradient is the sum of the derivatives of the basis functions multiplied by the neighboring values. Alternatively, if Hermite splines with explicit derivatives $m$ are used, the gradient components are derived from the $h_{10}, h_{11}$ terms scaled by $\Delta$ [S1, unofficial][S2]. In CGAL, `Interpolated_discrete_gradients_3` computes this analytically [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def catmull_rom(t, p0, p1, p2, p3):
    """
    Performs 1D Catmull-Rom interpolation.
    t: float in [0, 1]
    p0, p1, p2, p3: scalar values at indices -1, 0, 1, 2 relative to the interval
    """
    t2 = t * t
    t3 = t2 * t
    
    h0 = -0.5 * t3 + t2 - 0.5 * t
    h1 = 1.5 * t3 - 2.5 * t2 + 1.0
    h2 = -1.5 * t3 + 2.0 * t2 + 0.5 * t
    h3 = 0.5 * t3 - 0.5 * t2
    
    return h0 * p0 + h1 * p1 + h2 * p2 + h3 * p3

def interpolate_tricubic(p, grid, bounds_min, voxel_dim):
    """
    Computes scalar value at p using tri-cubic interpolation.
    p: tuple (px, py, pz)
    grid: 3D array of values
    bounds_min: tuple (xmin, ymin, zmin)
    voxel_dim: tuple (dx, dy, dz)
    """
    # 1. Voxel coordinates
    vx = int((p[0] - bounds_min[0]) / voxel_dim[0])
    vy = int((p[1] - bounds_min[1]) / voxel_dim[1])
    vz = int((p[2] - bounds_min[2]) / voxel_dim[2])
    
    # 2. Local offset
    tx = (p[0] - (bounds_min[0] + vx * voxel_dim[0])) / voxel_dim[0]
    ty = (p[1] - (bounds_min[1] + vy * voxel_dim[1])) / voxel_dim[1]
    tz = (p[2] - (bounds_min[2] + vz * voxel_dim[2])) / voxel_dim[2]
    
    # 3. Z-interpolation (4x4 slice)
    # Neighbors in Z: vz-1, vz, vz+1, vz+2
    # Neighbors in Y: vy-1, vy, vy+1, vy+2
    # Neighbors in X: vx-1, vx, vx+1, vx+2
    
    z_interp = [[0.0 for _ in range(4)] for _ in range(4)]
    
    for j in range(4): # j=0..3 corresponds to vy-1..vy+2
        for i in range(4): # i=0..3 corresponds to vx-1..vx+2
            # Get 4 Z neighbors
            p_z0 = grid[vx + i - 1][vy + j - 1][vz - 1]
            p_z1 = grid[vx + i - 1][vy + j - 1][vz]
            p_z2 = grid[vx + i - 1][vy + j - 1][vz + 1]
            p_z3 = grid[vx + i - 1][vy + j - 1][vz + 2]
            
            z_interp[j][i] = catmull_rom(tz, p_z0, p_z1, p_z2, p_z3)
            
    # 4. Y-interpolation (4 lines)
    y_interp = [0.0 for _ in range(4)]
    for i in range(4):
        y_interp[i] = catmull_rom(ty, z_interp[0][i], z_interp[1][i], z_interp[2][i], z_interp[3][i])
        
    # 5. X-interpolation
    result = catmull_rom(tx, y_interp[0], y_interp[1], y_interp[2], y_interp[3])
    
    return result
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The $4 \times 4 \times 4$ neighborhood extends 1 voxel outside the containing voxel. If the query point is near the grid boundary, indices may go out of bounds. Padding or clamping is required [S1, unofficial].
- **Precision:** Cubic interpolation can introduce overshoots (ringing artifacts) in regions with high-frequency content or sharp discontinuities [S1, unofficial].
- **Gradient Accuracy:** If gradients are not computed analytically from the basis, finite difference approximations may introduce noise [S2].

## Subtopics

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
Nearest Voxel Lookup retrieves the scalar value of the single grid voxel containing or nearest to a continuous sample point, without interpolating neighboring values. It solves the problem of mapping continuous world-space coordinates to discrete memory indices in a dense volume grid. Unlike [Interpolated Discrete Values & Gradients](1268-interpolated-discrete-values-gradients.md), which blends adjacent voxels for smoothness, this method produces a piecewise-constant field that preserves sharp discontinuities but introduces aliasing artifacts. It differs from [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md) by returning a single value rather than a set of corner indices, and from [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) by using zero-order approximation instead of higher-order continuity [G].

### [Dense Copy Extraction](3215-dense-copy-extraction.md)
Dense Copy Extraction extracts a spatial region defined by a bounding box from a sparse grid representation into a contiguous dense 3D array. It solves the problem of converting sparse, hierarchical data structures (like OpenVDB) into uniform grids suitable for dense rendering, machine learning inputs, or standard image processing. Unlike sibling methods that query individual points or gradients, this method performs a bulk conversion, initializing the output with a background value and overwriting it with interpolated values for all coordinates within the specified bounds.

### [Mipmap Grid Query](4154-mipmap-grid-query.md)
Mipmap Grid Query retrieves scalar values from a multi-resolution grid hierarchy by selecting a mip level based on the spatial distance of the query point from the center of the base bounding box. It solves the problem of aliasing and resolution mismatch in dense volume rendering by automatically downscaling the query resolution as the point moves further from the center, assuming a geometric scaling relationship between mip levels. It differs from standard trilinear interpolation by dynamically adjusting the grid resolution used for the lookup based on spatial context rather than a fixed user-defined level [G]. Builds on [Interpolated Discrete Values & Gradients](1268-interpolated-discrete-values-gradients.md).

### [Cylindrical Coordinate Interpolation](4282-cylindrical-coordinate-interpolation.md)
Cylindrical Coordinate Interpolation is a method for reconstructing volumetric data by mapping Cartesian sample points into cylindrical coordinates before sampling. It differs from its sibling, [Spherical Coordinate Interpolation](6278-spherical-coordinate-interpolation.md), by utilizing radial, angular, and vertical dimensions to handle data with cylindrical symmetry. It addresses the problem of accurately interpolating discrete polar points onto a Cartesian grid while preserving the global computational complexity of the reconstruction.

### [Data Interpolation](7180-data-interpolation.md)
Data Interpolation is a preprocessing technique that maps arbitrarily shaped or scattered 3D data grids into rectangular grids suitable for visualization software and volume texture generation. It solves the problem of uneven data structures by converting non-uniform or scattered inputs into regular grids, enabling shader programming and standard volume rendering pipelines.

### [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md)
Polynomial and Spline Interpolation provides a continuous, differentiable scalar or vector field from discrete grid data by evaluating weighted sums of basis functions over local support neighborhoods. It sets itself apart from [Nearest Voxel Lookup](1573-nearest-voxel-lookup-with-periodic-boundary.md) and [Dense Copy Extraction](3215-dense-copy-extraction.md) by ensuring $C^0$ or higher continuity, and from [Mipmap Grid Query](4154-mipmap-grid-query.md) by operating on a single resolution level with explicit basis definitions rather than pre-filtered hierarchical levels.

### [Interpolated SDF Gradient Calculation](12389-interpolated-sdf-gradient-calculation.md)
Calculates the gradient of a Signed Distance Field (SDF) at a continuous position within a 4x4x4 voxel block by interpolating gradients computed at the 8 corners of the block. It differs from nearest-neighbor gradient lookup by providing smooth, continuous derivatives essential for stable ray marching and normal estimation, while operating on a fixed local block size rather than a single voxel.

### Uncertain Gradient Field Interpolation
Reconstruction of the gradient field from an uncertain scalar field, resulting in a trivariate random vector $Y$ representing the uncertainties of partial derivatives along $x, y,$ and $z$ [S3]. For voxels falling out of the support of the basis function $\rho$, zero weights are assigned [S3]. The uncertain gradient is calculated as:
$$ \begin{bmatrix} Y_x \\ Y_y \\ Y_z \end{bmatrix} = \sum_i \begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} X_i $$
where weights $\begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} = \rho(\mathbf{v} - \mathbf{v}_i)$ [S3]. The basis function $\rho$ is commonly a combination of linear interpolation and central difference [S3]. When data uncertainties are modeled as uniform distributions, the joint distribution of $Y$ is a trivariate box spline [S3].

### Backprojection via Shepard's Interpolation
A technique to fit a continuous scale field $S(x)$ from a discrete set of LoG extrema $N=\{(x_i, t_i)\}$ by interpolating representative scales $t_i$ using basis functions [S4, unofficial]. The basis function $\Theta(d)$ uses the fourth-degree Wendland polynomial with compact support:
$$ \Theta(d) = \left[ \left[ 1 - \frac{d}{h} \right]_0^1 \right]^4 \left( 4\frac{d}{h} + 1 \right) $$
[S4, unofficial]. The scale field is $S(x) = \sum_{x_i \in N} \Theta(||x - x_i||) t_i$ [S4, unofficial]. Alternatively, the maximum operator can be used: $S(x) = \max_{x_i \in N} \{\Theta(||x - x_i||) t_i\}$ to prioritize larger features [S4, unofficial]. Parameter $h$ controls local support, typically $h = k \cdot t_i$ with $k \approx 1$ [S4, unofficial]. A pitfall is "bleeding," where small features inherit large scales, handled by a visibility test [S4, unofficial].

### Kriging Interpolation
An interpolation algorithm used to construct a regular grid data model from limited and discrete 3D borehole data [S5]. Steps include gridding, selecting reference points $p_i$, calculating coefficients $K, \lambda, D$ from a variation function, and solving the equation set using LU decomposition to get weights $\lambda_i$ [S5]. The estimated value is $R'(p_0) = \sum_{i=1}^n \lambda_i R(p_i)$ [S5].

### Zero Set Initialization of Isosurfaces
Initialization of closest points for isosurfaces defined on regularly sampled scalar fields using local gradient estimates instead of trilinear interpolation [S6]. It examines each volume cell; if the cell contains the isosurface, distance samples at corners are initialized with closest point information [S6]. The scalar field gradient $\nabla f(s)$ is estimated using central differencing [S6]. The linear approximation $f(s') = f(s) + \nabla f(s) \cdot (s' - s)$ is used to compute the closest point where $f(s') = f_0$ [S6]. This is inaccurate for high curvature regions [S6].

### Constant Filter Radius Mapping
Maps curvilinear grid data to a rectangular grid using a user-set constant filter radius for spatial interpolation [S7, unofficial]. A pitfall is that it violates the Nyquist criteria when input data has a wide range of scales and spatial densities, as a single radius cannot adequately sample all parts of the input grid [S7, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, as it accesses a fixed $4 \times 4 \times 4$ neighborhood (64 voxels) regardless of grid size [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the grid storage, where $N$ is the number of voxels [S1, unofficial].
- **Performance:** Tri-cubic interpolation is more computationally expensive than trilinear or nearest-neighbor lookup due to the 64 samples and cubic basis evaluations [S1, unofficial].
- **Strengths:** Provides smooth, continuous fields with continuous first derivatives (if gradients are computed correctly), reducing aliasing and visual artifacts in ray marching [S1, unofficial][S2].
- **Limitations:** Computationally heavier; prone to overshoot/ringing artifacts in high-contrast regions; requires careful boundary handling [S1, unofficial].
- **When to Use:** When smooth shading, accurate normals, or high-quality isosurface extraction is required [S2].
- **When Not to Use:** When performance is critical and visual smoothness is less important (use Nearest Voxel Lookup) or when data is inherently sparse (use Sparse Grid methods) [S1, unofficial][S2].

## Background
The method originates from the use of cubic spline interpolation for volume rendering and isosurfacing. It is implemented in libraries such as CGAL's `Isosurfacing_3` module, specifically in `Interpolated_discrete_values_3` and `Interpolated_discrete_gradients_3` [S2]. It is used in dual contouring and marching cubes algorithms to provide smooth field values and gradients for surface reconstruction [S2]. The technique is also applied in ray marching for volume rendering to ensure smooth transfer function evaluation [S1, unofficial].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S3] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
[S4] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
[S5] Volume Rendering of 3D Borehole Data Based on GPU, https://doi.org/10.2991/icsem.2013.181, fetched 2026-10-26
[S6] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409942/?q=volume data compression rendering, fetched 2026-10-28
[S7] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
