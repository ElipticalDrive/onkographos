# Interpolated Discrete Values & Gradients

Also called: Cubic Spline Interpolation, CubicSpline, Discrete Field Interpolation, Interpolated Grid Fields, Tri-Cubic Interpolation

## Overview
Interpolated Discrete Values & Gradients computes a continuous scalar field and its gradient at arbitrary 3D points by interpolating discrete values stored at grid vertices using tri-cubic (trilinear or cubic spline) basis functions [S1, unofficial][S2]. It solves the problem of mapping continuous world-space coordinates to discrete memory indices in a dense volume grid while providing smooth derivatives essential for ray marching and isosurfacing [S1, unofficial][S2]. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md), which produces a piecewise-constant field, this method blends adjacent voxels to ensure $C^0$ continuity (and $C^1$ if gradients are interpolated), reducing aliasing artifacts [S1, unofficial][S2]. It differs from [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) by providing the gradient as an integral part of the interpolation process rather than a post-hoc numerical derivative [S2].

## How it works
**Inputs:**
- A dense 3D grid of discrete scalar values $V$ at integer coordinates $(i, j, k)$.
- A query point $p = (p_x, p_y, p_z)$ in world space.
- Grid parameters: bounding box bounds, voxel dimensions (or spacing), and grid dimensions $(N_x, N_y, N_z)$ [S1, unofficial][S2].

**Outputs:**
- Scalar value $f(p)$ at the query point.
- Gradient vector $\nabla f(p) = (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})$ at the query point.

**Data Structures:**
- `Grid`: A 3D array or linearized memory block storing scalar values. Layout is typically row-major or column-major contiguous memory. Accessor `field(data, i, j, k)` retrieves the value at integer indices.
- `Values`: A field object wrapping the grid data, providing bounds checking and coordinate conversion [S2].
- `Gradients`: A field object storing or computing gradient vectors at grid vertices. If not pre-computed, it is derived from `Values` using finite differences or analytical derivatives if the source function is known [S2].

**Algorithm Steps:**

1.  **Coordinate Conversion:**
    Compute the integer voxel coordinates $v_i = (v_x, v_y, v_z)$ containing the query point $p$.
    $$ v_{axis} = \lfloor \frac{p_{axis} - bounds_{min, axis}}{voxel\_dim_{axis}} \rfloor $$
    where $axis \in \{x, y, z\}$ [S1, unofficial].

2.  **Local Offset Calculation:**
    Compute the minimum corner $v_{min}$ of the voxel containing $p$ and the fractional offset $t$ within the voxel.
    $$ v_{min, axis} = bounds_{min, axis} + v_{i, axis} \cdot voxel\_dim_{axis} $$
    $$ t_{axis} = \frac{p_{axis} - v_{min, axis}}{voxel\_dim_{axis}} $$
    The offset $t$ is in the range $[0, 1)$ [S1, unofficial].

3.  **Neighborhood Selection:**
    Tri-cubic interpolation requires a $4 \times 4 \times 4$ neighborhood of voxels. The indices relative to $v_i$ are $i \in \{-1, 0, 1, 2\}$, $j \in \{-1, 0, 1, 2\}$, $k \in \{-1, 0, 1, 2\}$ [S1, unofficial].
    *Edge Case:* If any neighbor index falls outside the grid boundaries, clamp the index to the nearest valid boundary or handle as an error depending on implementation [G].

4.  **1D Cubic Interpolation (Hermite Basis):**
    The core interpolation function `CINT_CR` (Cubic Interpolation with Continuous Derivatives) computes the value along one axis using two points and their derivatives (or finite difference approximations).
    
    Let the interval be $[x_k, x_{k+1}]$ with length $\Delta x = x_{k+1} - x_k$. Let $p_k, p_{k+1}$ be the values and $m_k, m_{k+1}$ be the derivatives (scaled by $\Delta x$ if necessary, or $m$ represents the derivative directly). The normalized parameter is $t = \frac{x - x_k}{\Delta x}$ [S1, unofficial].
    
    Compute powers:
    $$ t_2 = t^2, \quad t_3 = t^3 $$
    
    Compute basis functions (Hermite coefficients):
    $$ h_{00} = 2t^3 - 3t^2 + 1 $$
    $$ h_{10} = t^3 - 2t^2 + t $$
    $$ h_{01} = -2t^3 + 3t^2 $$
    $$ h_{11} = t^3 - t^2 $$
    
    Result:
    $$ \text{result} = h_{00} p_k + h_{10} \Delta x m_k + h_{01} p_{k+1} + h_{11} \Delta x m_{k+1} $$
    
    *Note:* If derivatives $m$ are not available at vertices, they are computed via central differences from the `Values` field: $m_k \approx \frac{V_{k+1} - V_{k-1}}{2 \Delta x}$ [S2, G].

5.  **Tri-Cubic Evaluation:**
    Perform interpolation sequentially along each axis.
    
    a. **Z-axis:** For each $j \in \{-1, 2\}$ and $i \in \{-1, 2\}$, interpolate along Z using 4 values:
       $$ z\_interp[j+1][i+1] = \text{CINT\_CR}(t_z, V_{v_x+i, v_y+j, v_z-1}, V_{v_x+i, v_y+j, v_z}, V_{v_x+i, v_y+j, v_z+1}, V_{v_x+i, v_y+j, v_z+2}) $$
       *Note:* The source code snippet in [S1, unofficial] uses indices $v_z-1$ to $v_z+2$. This implies the "derivative" inputs to `CINT_CR` are actually the values at the surrounding nodes if a standard cubic spline formulation is used where the derivative is approximated by the difference of neighbors, or the function signature in [S1, unofficial] abstracts the derivative calculation. Specifically, [S1, unofficial] shows `CINT_CR` taking 4 values. This corresponds to a Catmull-Rom spline or a cubic Hermite spline where the tangents are derived from the 4 points. The standard Catmull-Rom formula for point $P$ at parameter $t$ using $P_{-1}, P_0, P_1, P_2$ is:
       $$ P(t) = \frac{1}{2} \left( (2P_0) + (-P_{-1} + P_1)t + (2P_{-1} - 5P_0 + 4P_1 - P_2)t^2 + (-P_{-1} + 3P_0 - 3P_1 + P_2)t^3 \right) $$
       The source [S1, unofficial] explicitly lists `CINT_CR` arguments as 4 values. We implement `CINT_CR` as Catmull-Rom interpolation [S1, G, unofficial].

    b. **Y-axis:** For each $i \in \{-1, 2\}$, interpolate the Z-interpolated results along Y:
       $$ y\_interp[i] = \text{CINT\_CR}(t_y, z\_interp[0][i], z\_interp[1][i], z\_interp[2][i], z\_interp[3][i]) $$

    c. **X-axis:** Interpolate the Y-interpolated results along X:
       $$ f(p) = \text{CINT\_CR}(t_x, y\_interp[0], y\_interp[1], y\_interp[2], y\_interp[3]) $$

6.  **Gradient Computation:**
    The gradient $\nabla f(p)$ can be computed analytically if the basis functions' derivatives are known, or by interpolating pre-computed vertex gradients.
    
    *Analytical Method:* Differentiate the basis functions $h_{00}, h_{10}, h_{01}, h_{11}$ with respect to $t$, then scale by $1/\Delta x$ for each axis.
    $$ \frac{\partial f}{\partial x} = \frac{1}{\Delta x} \frac{\partial f}{\partial t_x} $$
    
    *Discrete Method:* If `Gradients` field is populated with vertex gradients $G_{i,j,k}$, perform the same tri-cubic interpolation structure but on the gradient components $G_x, G_y, G_z$ separately [S2].
    
    $$ \nabla f(p) = \text{TriCubicInterpolate}(Gradients, p) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def catmull_rom(t, p0, p1, p2, p3):
    """
    Catmull-Rom interpolation for a single axis.
    t is in [0, 1].
    p0, p1, p2, p3 are values at indices -1, 0, 1, 2 relative to the interval [0, 1].
    """
    t2 = t * t
    t3 = t2 * t
    # Basis functions for Catmull-Rom
    # P(t) = 0.5 * ((2*p1) + (-p0 + p2)*t + (2*p0 - 5*p1 + 4*p2 - p3)*t^2 + (-p0 + 3*p1 - 3*p2 + p3)*t^3)
    term1 = 2.0 * p1
    term2 = (-p0 + p2) * t
    term3 = (2.0 * p0 - 5.0 * p1 + 4.0 * p2 - p3) * t2
    term4 = (-p0 + 3.0 * p1 - 3.0 * p2 + p3) * t3
    return 0.5 * (term1 + term2 + term3 + term4)

def tri_cubic_interpolate(grid, p, bounds_min, voxel_dim, grid_dims):
    """
    Computes scalar value at p using tri-cubic interpolation.
    """
    # 1. Compute voxel coordinates
    v_x = int((p.x - bounds_min.x) / voxel_dim.x)
    v_y = int((p.y - bounds_min.y) / voxel_dim.y)
    v_z = int((p.z - bounds_min.z) / voxel_dim.z)
    
    # 2. Compute local offset t
    t_x = (p.x - (bounds_min.x + v_x * voxel_dim.x)) / voxel_dim.x
    t_y = (p.y - (bounds_min.y + v_y * voxel_dim.y)) / voxel_dim.y
    t_z = (p.z - (bounds_min.z + v_z * voxel_dim.z)) / voxel_dim.z
    
    # 3. Fetch 4x4x4 neighborhood
    # Indices relative to v: -1, 0, 1, 2
    # We need to handle boundary clamping here [G]
    def get_val(ix, iy, iz):
        # Clamp indices to [0, grid_dims-1]
        ix = max(0, min(ix, grid_dims.x - 1))
        iy = max(0, min(iy, grid_dims.y - 1))
        iz = max(0, min(iz, grid_dims.z - 1))
        return grid[ix, iy, iz]

    # 4. Interpolate along Z for each (i, j)
    z_interp = [[0.0 for _ in range(4)] for _ in range(4)]
    for i in range(-1, 3):
        for j in range(-1, 3):
            # Neighbors in Z: v_z-1, v_z, v_z+1, v_z+2
            p0 = get_val(v_x + i, v_y + j, v_z - 1)
            p1 = get_val(v_x + i, v_y + j, v_z)
            p2 = get_val(v_x + i, v_y + j, v_z + 1)
            p3 = get_val(v_x + i, v_y + j, v_z + 2)
            z_interp[j + 1][i + 1] = catmull_rom(t_z, p0, p1, p2, p3)

    # 5. Interpolate along Y for each i
    y_interp = [0.0 for _ in range(4)]
    for i in range(-1, 3):
        # Neighbors in Y: v_y-1, v_y, v_y+1, v_y+2
        p0 = z_interp[0][i + 1]
        p1 = z_interp[1][i + 1]
        p2 = z_interp[2][i + 1]
        p3 = z_interp[3][i + 1]
        y_interp[i + 1] = catmull_rom(t_y, p0, p1, p2, p3)

    # 6. Interpolate along X
    p0 = y_interp[0]
    p1 = y_interp[1]
    p2 = y_interp[2]
    p3 = y_interp[3]
    return catmull_rom(t_x, p0, p1, p2, p3)
```

## Subtopics

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
Nearest Voxel Lookup retrieves the scalar value of the single grid voxel containing or nearest to a continuous sample point, without interpolating neighboring values. It solves the problem of mapping continuous world-space coordinates to discrete memory indices in a dense volume grid. Unlike [Interpolated Discrete Values & Gradients](1268-interpolated-discrete-values-gradients.md), which blends adjacent voxels for smoothness, this method produces a piecewise-constant field that preserves sharp discontinuities but introduces aliasing artifacts. It differs from [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md) by returning a single value rather than a set of corner indices, and from [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) by using zero-order approximation instead of higher-order continuity [G].

### [Dense Copy Extraction](3215-dense-copy-extraction.md)
Dense Copy Extraction extracts a spatial region defined by a bounding box from a sparse grid representation into a contiguous dense 3D array. It solves the problem of converting sparse, hierarchical data structures (like OpenVDB) into uniform grids suitable for dense rendering, machine learning inputs, or standard image processing. Unlike sibling methods that query individual points or gradients, this method performs a bulk conversion, initializing the output with a background value and overwriting it with interpolated values for all coordinates within the specified bounds.

### [Mipmap Grid Query](4154-mipmap-grid-query.md)
Mipmap Grid Query retrieves scalar values from a multi-resolution grid hierarchy by selecting a mip level based on the spatial distance of the query point from the center of the base bounding box. It solves the problem of aliasing and resolution mismatch in dense volume rendering by automatically downscaling the query resolution as the point moves further from the center, assuming a geometric scaling relationship between mip levels. It differs from standard trilinear interpolation by dynamically adjusting the grid resolution used for the lookup based on spatial context rather than a fixed user-defined level.

### [Cylindrical Coordinate Interpolation](4282-cylindrical-coordinate-interpolation.md)
Cylindrical Coordinate Interpolation maps Cartesian query points to cylindrical space before sampling the volume data, enabling efficient texture-based lookup for geometries with cylindrical symmetry. It differs from standard Cartesian interpolation by remapping the coordinate system to align with the geometry's principal axis.

### [Data Interpolation](7180-data-interpolation.md)
Data Interpolation is a preprocessing technique that maps arbitrarily shaped or scattered 3D data grids into rectangular grids suitable for visualization software and volume texture generation. It solves the problem of uneven data structures by converting non-uniform or scattered inputs into regular grids, enabling shader programming and standard volume rendering pipelines. It differs from sibling methods like [Nearest Voxel Lookup](1573-nearest-voxel-lookup-with-periodic-boundary.md) by actively reconstructing values via filtering rather than simple lookup, and from [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md) by focusing on the global grid construction and filtering strategy rather than local basis functions alone.

### [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md)
Polynomial and Spline Interpolation provides scalar value and gradient fields computed from discrete data at grid vertices using continuous basis functions, enabling smooth evaluation at arbitrary coordinates. It is distinguished from its parent [Interpolated Discrete Values & Gradients](1268-interpolated-discrete-values-gradients.md) by its explicit use of higher-order continuity (e.g., $C^1$ or $C^2$) and local support properties, and from its sibling [Data Interpolation](7180-data-interpolation.md) by its focus on piecewise polynomial bases (Hermite, B-splines) rather than general kernel methods.

### [Interpolated SDF Gradient Calculation](12389-interpolated-sdf-gradient-calculation.md)
Calculates the gradient of a Signed Distance Field (SDF) at a continuous position within a 4x4x4 voxel block by interpolating gradients computed at the 8 corners of the block. It differs from nearest-neighbor gradient lookup by providing smooth, continuous derivatives essential for stable ray marching and normal estimation, while operating on a fixed local block size rather than a single voxel.

### Uncertain Gradient Field Interpolation
Reconstruction of the gradient field from an uncertain scalar field, resulting in a trivariate random vector $Y$ representing the uncertainties of partial derivatives along x, y, and z [S3]. For voxels falling out of the support of the basis function $\rho$, zero weights are assigned [S3]. The uncertain gradient is computed as $\begin{bmatrix} Y_x \\ Y_y \\ Y_z \end{bmatrix} = \sum_i \begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} X_i$ with weights $\begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} = \rho(\mathbf{v} - \mathbf{v}_i)$, where $\rho$ is the basis function (derivative reconstruction filter) and $X_i$ are random variables at neighboring voxels [S3]. When data uncertainties are modeled as (scaled) uniform distributions, the distribution of each partial derivative is a box spline [S3]. The basis function $\rho$ is commonly a combination of linear interpolation and central difference [S3].

### Backprojection via Shepard's Interpolation
A technique to fit a continuous scale field $S(x)$ from a discrete set of LoG extrema $N=\{(x_i, t_i)\}$ by interpolating the representative scales $t_i$ using basis functions, allowing for a smooth transfer function [S4, unofficial]. The basis function $\Theta(d)$ uses the fourth-degree Wendland polynomial with compact support: $\Theta(d) = [1 - d/h]_0^1)^4 (4d/h + 1)$, where $[\cdot]_0^1$ is a clamping function between 0 and 1 [S4, unofficial]. The scale field is $S(x) = \sum_{x_i \in N} \Theta(||x - x_i||) t_i$ [S4, unofficial]. Alternatively, the maximum operator $S(x) = \max_{x_i \in N} \{\Theta(||x - x_i||) t_i\}$ can be used to give priority to larger features [S4, unofficial]. The parameter $h$ controls local support, typically $h = k \cdot t_i$ with $k=1$ [S4, unofficial]. A pitfall is "bleeding," where a small feature inherits the scale of a large feature, handled by a visibility test [S4, unofficial].

### Kriging Interpolation
An interpolation algorithm used to construct a regular grid data model from limited and discrete 3D borehole data [S5]. Steps include: (1) Gridding: Set grid range and size; (2) Calculate estimated point coordinates $p_0$; (3) Select reference points $p_i$ via search strategy; (4) Calculate coefficients $K, \lambda, D$ of the equation set according to the variation function; (5) Solve equation set using LU method to get weight coefficient $\lambda_i$; (6) Round result $R'$ to $R$; (7) Repeat until all grid points are found [S5]. The formula is $R'(p_0) = \sum_{i=1}^n \lambda_i R(p_i)$, where $n$ is the number of interpolation points [S5].

### Zero Set Initialization of Isosurfaces
Initialization of closest points for isosurfaces defined on regularly sampled scalar fields using local gradient estimates instead of trilinear interpolation [S6]. The algorithm examines each volume cell; if it contains the isosurface, distance samples at cell corners are initialized with closest point information [S6]. The scalar field gradient $\nabla f(s)$ is estimated using central differencing [S6]. The linear approximation is $f(s') = f(s) + \nabla f(s) \cdot (s' - s)$, and the closest point is computed by solving for $s'$ where $f(s') = f_0$ (isovalue) [S6]. This approximation is inaccurate for high curvature regions but efficient; accuracy can be improved by Newton iterations [S6].

### Constant Filter Radius Mapping
Maps curvilinear grid data to a rectangular grid using a user-set constant filter radius for spatial interpolation [S7, unofficial]. A pitfall is that it violates the Nyquist criteria when input data has a wide range of scales and spatial densities; a single radius cannot adequately sample all parts of the input grid [S7, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, as it accesses a fixed $4 \times 4 \times 4 = 64$ voxel neighborhood [S1, unofficial].
- **Memory Complexity:** $O(N_x N_y N_z)$ for the grid storage [S2].
- **Performance:** Requires 64 texture fetches or memory loads per sample point for tri-cubic interpolation [S1, unofficial]. This is more expensive than trilinear (8 fetches) or nearest-neighbor (1 fetch) but provides smoother results [S1, unofficial].
- **Strengths:** Provides $C^1$ continuity (if gradients are interpolated) or $C^0$ continuity (if only values are interpolated with Catmull-Rom), reducing aliasing and providing smooth normals for ray marching [S1, unofficial][S2].
- **Limitations:** Higher computational cost than lower-order interpolations. Boundary handling requires clamping or periodic extension, which can introduce artifacts at edges [G].
- **When to use:** High-quality volume rendering, isosurfacing (marching cubes, dual contouring) where smooth gradients are required [S2].
- **When not to use:** Real-time rendering with strict performance constraints where trilinear or nearest-neighbor is sufficient, or when sharp discontinuities must be preserved without blurring [S1, unofficial][S2].

## Background
The method originates from the need to evaluate continuous fields from discrete samples in computer graphics and scientific visualization. It is implemented in libraries such as CGAL's Isosurfacing module (`CGAL::Isosurfacing::Interpolated_discrete_values_3` and `CGAL::Isosurfacing::Interpolated_discrete_gradients_3`) for isosurface extraction from discrete data [S2]. The specific cubic spline formulation using 4 points (Catmull-Rom) is a standard technique in computer graphics for smooth interpolation [S1, unofficial]. Variants like uncertain gradient interpolation extend the method to handle stochastic fields [S3], and Shepard's interpolation is used for scale-based transfer functions [S4, unofficial].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S3] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
[S4] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
[S5] Volume Rendering of 3D Borehole Data Based on GPU, https://doi.org/10.2991/icsem.2013.181, fetched 2026-10-26
[S6] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409942/?q=volume data compression rendering, fetched 2026-10-28
[S7] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
