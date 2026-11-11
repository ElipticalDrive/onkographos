# Interpolated Discrete Values & Gradients

Also called: Cubic Spline Interpolation, CubicSpline, Discrete Field Interpolation, Interpolated Grid Fields, Tri-Cubic Interpolation

## Overview
Interpolated Discrete Values & Gradients computes a continuous scalar field and its spatial gradient at arbitrary 3D points by interpolating discrete values stored at grid vertices using tri-cubic (tensor-product cubic) interpolation [S1, unofficial][S2]. It solves the problem of mapping continuous world-space coordinates to discrete memory indices while providing $C^1$-continuous derivatives, which are essential for stable ray marching, normal estimation, and isosurface extraction [S1, unofficial][S2]. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md) which produces piecewise-constant fields with aliasing, or [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md) which only gathers corner data for lower-order methods, this method explicitly blends a 4x4x4 neighborhood of voxels to ensure smoothness and derivative continuity [S1, unofficial][S2].

## How it works
**Inputs:**
- A query point $p = (p_x, p_y, p_z)$ in world space.
- A dense 3D grid of scalar values $V$ with dimensions $D_x \times D_y \times D_z$, stored in a contiguous array or tensor.
- Grid bounding box bounds $[b_{min}, b_{max}]$ and voxel dimensions $dx, dy, dz$.

**Outputs:**
- Scalar value $f(p)$ at the query point.
- Gradient vector $\nabla f(p) = (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})$ at the query point.

**Data Structures:**
- The grid is a 3D array `grid[x][y][z]` of type `float` or `double`.
- Indices are 0-based. The voxel at integer coordinates $(i, j, k)$ corresponds to the cell corner at world position $b_{min} + (i \cdot dx, j \cdot dy, k \cdot dz)$.

**Algorithm Steps:**

1.  **Coordinate Mapping:**
    Compute the integer voxel coordinates containing the query point $p$:
    $$ v_i[x] = \lfloor \frac{p_x - b_{min,x}}{dx} \rfloor $$
    $$ v_i[y] = \lfloor \frac{p_y - b_{min,y}}{dy} \rfloor $$
    $$ v_i[z] = \lfloor \frac{p_z - b_{min,z}}{dz} \rfloor $$
    [G] Clamp $v_i$ to valid grid indices $[0, D_x-1], [0, D_y-1], [0, D_z-1]$ to handle boundary queries. If $p$ is outside the bounding box, return a default value (e.g., 0 or background) and zero gradient.

2.  **Fractional Offset:**
    Compute the local offset $t$ within the voxel, normalized to $[0, 1]$:
    $$ v_{min} = b_{min} + v_i \cdot (dx, dy, dz) $$
    $$ t_x = \frac{p_x - v_{min,x}}{dx} $$
    $$ t_y = \frac{p_y - v_{min,y}}{dy} $$
    $$ t_z = \frac{p_z - v_{min,z}}{dz} $$
    [G] Clamp $t$ to $[0, 1]$ to handle floating-point precision errors near boundaries.

3.  **Tri-Cubic Interpolation (Scalar Value):**
    Tri-cubic interpolation requires a 4x4x4 neighborhood of values centered around the voxel containing $p$. The interpolation is performed as three sequential 1D cubic spline interpolations along the z, y, and x axes.

    Let $v_i = (v_x, v_y, v_z)$. The indices for the 4x4x4 block range from $v_x-1$ to $v_x+2$ along x, $v_y-1$ to $v_y+2$ along y, and $v_z-1$ to $v_z+2$ along z.

    **Step 3a: Interpolate along Z-axis.**
    For each $j \in [-1, 2]$ and $i \in [-1, 2]$, compute the interpolated value along the z-axis using the 4 voxels at $(v_x+i, v_y+j, v_z+k)$ for $k \in [-1, 2]$:
    $$ z\_interp[j+1][i+1] = \text{CubicSpline1D}(t_z, V[v_x+i][v_y+j][v_z-1], V[v_x+i][v_y+j][v_z], V[v_x+i][v_y+j][v_z+1], V[v_x+i][v_y+j][v_z+2]) $$
    [S1, unofficial]

    **Step 3b: Interpolate along Y-axis.**
    For each $i \in [-1, 2]$, interpolate the results from Step 3a along the y-axis:
    $$ y\_interp[i] = \text{CubicSpline1D}(t_y, z\_interp[0][i], z\_interp[1][i], z\_interp[2][i], z\_interp[3][i]) $$
    [S1, unofficial]

    **Step 3c: Interpolate along X-axis.**
    Interpolate the results from Step 3b along the x-axis to get the final scalar value:
    $$ f(p) = \text{CubicSpline1D}(t_x, y\_interp[0], y\_interp[1], y\_interp[2], y\_interp[3]) $$
    [S1, unofficial]

4.  **Gradient Computation:**
    The gradient is computed by applying the same tri-cubic interpolation logic to the derivatives of the basis functions. Alternatively, if discrete gradients are precomputed at vertices, they are interpolated similarly [S2]. If not precomputed, they can be derived analytically from the scalar field interpolation weights or computed via finite differences on the discrete values [S2].

    **Analytical Gradient via Basis Functions:**
    The 1D cubic Hermite basis functions for an interval $[x_k, x_{k+1}]$ with values $p_k, p_{k+1}$ and derivatives $m_k, m_{k+1}$ (where $m$ represents the derivative scaled by the interval length $\Delta x$) are:
    $$ h_{00}(t) = 2t^3 - 3t^2 + 1 $$
    $$ h_{10}(t) = t^3 - 2t^2 + t $$
    $$ h_{01}(t) = -2t^3 + 3t^2 $$
    $$ h_{11}(t) = t^3 - t^2 $$
    [S1, unofficial]

    The interpolated value is:
    $$ S(t) = h_{00}(t)p_k + h_{10}(t)\Delta x m_k + h_{01}(t)p_{k+1} + h_{11}(t)\Delta x m_{k+1} $$
    [S1, unofficial]

    The derivative with respect to the local parameter $t$ is:
    $$ \frac{dS}{dt} = (6t^2 - 6t)p_k + (3t^2 - 4t + 1)\Delta x m_k + (-6t^2 + 6t)p_{k+1} + (3t^2 - 2t)\Delta x m_{k+1} $$
    The physical derivative is $\frac{dS}{dx} = \frac{1}{\Delta x} \frac{dS}{dt}$.

    To compute $\nabla f(p)$, perform the tri-cubic interpolation structure again, but propagate the derivatives.
    1.  Compute $z\_derivs[j+1][i+1]$ by differentiating the 1D cubic spline along Z. This requires the discrete gradients $\frac{\partial V}{\partial z}$ at the 4x4x4 corners. If only scalar values are available, estimate $\frac{\partial V}{\partial z}$ using central differences on the grid vertices.
    2.  Compute $y\_derivs[i]$ by differentiating the interpolation along Y.
    3.  Compute $x\_deriv$ by differentiating the interpolation along X.
    4.  $\nabla f(p) = (x\_deriv, y\_deriv, z\_deriv)$.

    [G] If discrete gradients are not stored, a common approximation for the gradient at a vertex $(i,j,k)$ is:
    $$ \frac{\partial V}{\partial x} \approx \frac{V[i+1][j][k] - V[i-1][j][k]}{2dx} $$
    This central difference is used to populate the derivative inputs for the tri-cubic gradient calculation.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_spline_1d(t, p0, p1, m0, m1, delta):
    """
    1D Cubic Hermite Interpolation.
    t: normalized parameter in [0, 1]
    p0, p1: values at endpoints
    m0, m1: derivatives at endpoints (scaled by delta, i.e., m = df/dx * delta)
    delta: length of the interval
    Returns: interpolated value and derivative df/dx
    """
    t2 = t * t
    t3 = t2 * t
    
    # Basis functions for value
    h00 = 2 * t3 - 3 * t2 + 1
    h10 = t3 - 2 * t2 + t
    h01 = -2 * t3 + 3 * t2
    h11 = t3 - t2
    
    # Interpolated value
    val = h00 * p0 + h10 * m0 + h01 * p1 + h11 * m1
    
    # Derivative with respect to t
    dh00_dt = 6 * t2 - 6 * t
    dh10_dt = 3 * t2 - 4 * t + 1
    dh01_dt = -6 * t2 + 6 * t
    dh11_dt = 3 * t2 - 2 * t
    
    dval_dt = dh00_dt * p0 + dh10_dt * m0 + dh01_dt * p1 + dh11_dt * m1
    
    # Derivative with respect to x
    dval_dx = dval_dt / delta
    
    return val, dval_dx

def interpolate_tricubic(grid, bounds, voxel_dims, p):
    """
    Interpolates scalar value and gradient at point p using tri-cubic interpolation.
    grid: 3D array [x][y][z]
    bounds: [min_x, min_y, min_z, max_x, max_y, max_z]
    voxel_dims: [dx, dy, dz]
    p: [px, py, pz]
    """
    dx, dy, dz = voxel_dims
    min_x, min_y, min_z = bounds[0], bounds[1], bounds[2]
    
    # 1. Compute voxel coordinates
    ix = int((p[0] - min_x) / dx)
    iy = int((p[1] - min_y) / dy)
    iz = int((p[2] - min_z) / dz)
    
    # Clamp indices
    ix = max(0, min(ix, grid.shape[0] - 2))
    iy = max(0, min(iy, grid.shape[1] - 2))
    iz = max(0, min(iz, grid.shape[2] - 2))
    
    # 2. Compute fractional offset
    t_x = (p[0] - (min_x + ix * dx)) / dx
    t_y = (p[1] - (min_y + iy * dy)) / dy
    t_z = (p[2] - (min_z + iz * dz)) / dz
    
    # Clamp t
    t_x = max(0.0, min(1.0, t_x))
    t_y = max(0.0, min(1.0, t_y))
    t_z = max(0.0, min(1.0, t_z))
    
    # 3. Precompute discrete gradients at vertices using central differences
    # This is a helper to get m0, m1 for each axis. 
    # Note: In a full implementation, these might be precomputed or cached.
    def get_gradient_at_vertex(x, y, z):
        # Central difference for x
        gx = (grid[x+1][y][z] - grid[x-1][y][z]) / (2 * dx)
        # Central difference for y
        gy = (grid[x][y+1][z] - grid[x][y-1][z]) / (2 * dy)
        # Central difference for z
        gz = (grid[x][y][z+1] - grid[x][y][z-1]) / (2 * dz)
        return gx, gy, gz

    # 4. Tri-Cubic Interpolation
    # We need to interpolate along Z, then Y, then X.
    # For each step, we need values and derivatives.
    
    # Step 4a: Interpolate along Z
    # z_interp[j][i] will store (value, d/dz)
    z_interp = [[None for _ in range(4)] for _ in range(4)]
    
    for j in range(-1, 3): # j from -1 to 2
        for i in range(-1, 3): # i from -1 to 2
            # Indices for the 4 points along Z
            idx_x = ix + i
            idx_y = iy + j
            idx_z_start = iz - 1
            
            # Values
            p0 = grid[idx_x][idx_y][idx_z_start]
            p1 = grid[idx_x][idx_y][idx_z_start + 1]
            p2 = grid[idx_x][idx_y][idx_z_start + 2]
            p3 = grid[idx_x][idx_y][idx_z_start + 3]
            
            # Derivatives at these points (d/dz)
            # We need the gradient at each of the 4 points
            g0 = get_gradient_at_vertex(idx_x, idx_y, idx_z_start)[2]
            g1 = get_gradient_at_vertex(idx_x, idx_y, idx_z_start + 1)[2]
            g2 = get_gradient_at_vertex(idx_x, idx_y, idx_z_start + 2)[2]
            g3 = get_gradient_at_vertex(idx_x, idx_y, idx_z_start + 3)[2]
            
            # Interpolate between p0/p1 using t_z
            # But tri-cubic uses 4 points. The standard cubic spline uses 2 points and their derivatives.
            # The source [S1, unofficial] implies a 4-point stencil for the *value* interpolation, 
            # but the formula provided is for Hermite (2 points, 2 derivatives).
            # Re-reading [S1, unofficial]: "CINT_CR(t.z, field(data, v_i[0]+i, v_i[1]+j, v_i[2]-1), ...)"
            # This suggests a 4-point cubic interpolation (like Catmull-Rom or B-spline) 
            # rather than Hermite. However, the basis functions h00..h11 are Hermite.
            # Standard Tri-Cubic in graphics often uses Catmull-Rom which is a specific Hermite spline.
            # Let's assume the source [S1, unofficial] uses a 4-point Catmull-Rom style interpolation 
            # which can be converted to Hermite form, OR it uses the 4 points to estimate derivatives.
            
            # Given the explicit basis functions h00..h11 in [S1, unofficial], it is Hermite.
            # The 4 inputs to CINT_CR in [S1, unofficial] are likely: p_{i-1}, p_i, p_{i+1}, p_{i+2}.
            # Catmull-Rom converts these to derivatives: m_i = 0.5 * (p_{i+1} - p_{i-1}).
            
            # Let's implement Catmull-Rom to Hermite conversion for the Z-axis step.
            # For point p1 (at index 0 in the local interval for t_z=0), 
            # the derivative m0 = 0.5 * (p2 - p0)
            # The derivative m1 = 0.5 * (p3 - p1)
            
            m0_z = 0.5 * (p2 - p0)
            m1_z = 0.5 * (p3 - p1)
            
            val_z, dval_z_dt = cubic_spline_1d(t_z, p1, p2, m0_z * dz, m1_z * dz, dz)
            dval_z_dx = dval_z_dt / dz
            
            z_interp[j+1][i+1] = (val_z, dval_z_dx)

    # Step 4b: Interpolate along Y
    y_interp = [None for _ in range(4)]
    for i in range(4):
        val_z0, dval_z0 = z_interp[0][i]
        val_z1, dval_z1 = z_interp[1][i]
        val_z2, dval_z2 = z_interp[2][i]
        val_z3, dval_z3 = z_interp[3][i]
        
        # Catmull-Rom derivatives for Y
        m0_y = 0.5 * (val_z2 - val_z0)
        m1_y = 0.5 * (val_z3 - val_z1)
        
        val_y, dval_y_dt = cubic_spline_1d(t_y, val_z1, val_z2, m0_y * dy, m1_y * dy, dy)
        dval_y_dy = dval_y_dt / dy
        
        y_interp[i] = (val_y, dval_y_dy)

    # Step 4c: Interpolate along X
    val_x0, dval_x0 = y_interp[0]
    val_x1, dval_x1 = y_interp[1]
    val_x2, dval_x2 = y_interp[2]
    val_x3, dval_x3 = y_interp[3]
    
    m0_x = 0.5 * (val_x2 - val_x0)
    m1_x = 0.5 * (val_x3 - val_x1)
    
    final_val, dval_x_dt = cubic_spline_1d(t_x, val_x1, val_x2, m0_x * dx, m1_x * dx, dx)
    final_dx = dval_x_dt / dx
    
    # The gradient components are:
    # dx: final_dx
    # dy: dval_y_dy (from the last Y interpolation step, but we need to propagate the derivative through X?)
    # dz: dval_z_dx (from the last Z interpolation step?)
    
    # Correction: The gradient calculation in tri-cubic is complex because the derivative 
    # of the final result with respect to Y depends on the derivative of the intermediate Y interpolation.
    # The method above computes the derivative of the *intermediate* values.
    # To get the true gradient, we must differentiate the final composition.
    # However, a simpler and common approach in implementations like CGAL [S2] is to 
    # interpolate the pre-computed vertex gradients using the same basis functions.
    
    # Let's stick to the CGAL approach [S2]: Interpolate the gradients.
    # We need to re-run the interpolation structure but passing gradient components.
    
    # Re-compute Z-interpolated gradients
    z_grad_interp = [[None for _ in range(4)] for _ in range(4)]
    for j in range(-1, 3):
        for i in range(-1, 3):
            idx_x = ix + i
            idx_y = iy + j
            idx_z_start = iz - 1
            
            # Get full gradient vectors at the 4 vertices along Z
            g0 = get_gradient_at_vertex(idx_x, idx_y, idx_z_start)
            g1 = get_gradient_at_vertex(idx_x, idx_y, idx_z_start + 1)
            g2 = get_gradient_at_vertex(idx_x, idx_y, idx_z_start + 2)
            g3 = get_gradient_at_vertex(idx_x, idx_y, idx_z_start + 3)
            
            # Interpolate each component of the gradient along Z using Catmull-Rom
            # For Gx
            m0_gx = 0.5 * (g2[0] - g0[0])
            m1_gx = 0.5 * (g3[0] - g1[0])
            val_gx, _ = cubic_spline_1d(t_z, g1[0], g2[0], m0_gx * dz, m1_gx * dz, dz)
            
            # For Gy
            m0_gy = 0.5 * (g2[1] - g0[1])
            m1_gy = 0.5 * (g3[1] - g1[1])
            val_gy, _ = cubic_spline_1d(t_z, g1[1], g2[1], m0_gy * dz, m1_gy * dz, dz)
            
            # For Gz
            m0_gz = 0.5 * (g2[2] - g0[2])
            m1_gz = 0.5 * (g3[2] - g1[2])
            val_gz, _ = cubic_spline_1d(t_z, g1[2], g2[2], m0_gz * dz, m1_gz * dz, dz)
            
            z_grad_interp[j+1][i+1] = (val_gx, val_gy, val_gz)

    # Re-compute Y-interpolated gradients
    y_grad_interp = [None for _ in range(4)]
    for i in range(4):
        g0 = z_grad_interp[0][i]
        g1 = z_grad_interp[1][i]
        g2 = z_grad_interp[2][i]
        g3 = z_grad_interp[3][i]
        
        # Interpolate Gx along Y
        m0_gx = 0.5 * (g2[0] - g0[0])
        m1_gx = 0.5 * (g3[0] - g1[0])
        val_gx, _ = cubic_spline_1d(t_y, g1[0], g2[0], m0_gx * dy, m1_gx * dy, dy)
        
        # Interpolate Gy along Y
        m0_gy = 0.5 * (g2[1] - g0[1])
        m1_gy = 0.5 * (g3[1] - g1[1])
        val_gy, _ = cubic_spline_1d(t_y, g1[1], g2[1], m0_gy * dy, m1_gy * dy, dy)
        
        # Interpolate Gz along Y
        m0_gz = 0.5 * (g2[2] - g0[2])
        m1_gz = 0.5 * (g3[2] - g1[2])
        val_gz, _ = cubic_spline_1d(t_y, g1[2], g2[2], m0_gz * dy, m1_gz * dy, dy)
        
        y_grad_interp[i] = (val_gx, val_gy, val_gz)

    # Re-compute X-interpolated gradients
    g0 = y_grad_interp[0]
    g1 = y_grad_interp[1]
    g2 = y_grad_interp[2]
    g3 = y_grad_interp[3]
    
    # Interpolate Gx along X
    m0_gx = 0.5 * (g2[0] - g0[0])
    m1_gx = 0.5 * (g3[0] - g1[0])
    final_gx, _ = cubic_spline_1d(t_x, g1[0], g2[0], m0_gx * dx, m1_gx * dx, dx)
    
    # Interpolate Gy along X
    m0_gy = 0.5 * (g2[1] - g0[1])
    m1_gy = 0.5 * (g3[1] - g1[1])
    final_gy, _ = cubic_spline_1d(t_x, g1[1], g2[1], m0_gy * dx, m1_gy * dx, dx)
    
    # Interpolate Gz along X
    m0_gz = 0.5 * (g2[2] - g0[2])
    m1_gz = 0.5 * (g3[2] - g1[2])
    final_gz, _ = cubic_spline_1d(t_x, g1[2], g2[2], m0_gz * dx, m1_gz * dx, dx)
    
    return final_val, (final_gx, final_gy, final_gz)
```

## Subtopics

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Extracts the 8 vertices of the enclosing grid cell. Foundational for trilinear interpolation.

### [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md)
Single-voxel lookup, piecewise-constant.

### [Dense Copy Extraction](3215-dense-copy-extraction.md)
Bulk conversion from sparse to dense.

### [Mipmap Grid Query](4154-mipmap-grid-query.md)
Multi-resolution query.

### [Cylindrical Coordinate Interpolation](4282-cylindrical-coordinate-interpolation.md)
Cylindrical symmetry.

### [Data Interpolation](7180-data-interpolation.md)
Global grid construction.

### [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md)
General higher-order continuity.

### [Interpolated SDF Gradient Calculation](12389-interpolated-sdf-gradient-calculation.md)
Gradient of SDF in 4x4x4 block.

### Uncertain Gradient Field Interpolation
Reconstructs the gradient field from an uncertain scalar field, resulting in a trivariate random vector $Y$ representing the uncertainties of partial derivatives along x, y, and z [S3]. The uncertain gradient $Y$ is obtained from linear combinations of random variables $X_i$ at neighboring voxels: $\begin{bmatrix} Y_x \\ Y_y \\ Y_z \end{bmatrix} = \sum_i \begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} X_i$ with weights $\begin{bmatrix} \alpha_i \\ \beta_i \\ \gamma_i \end{bmatrix} = \rho(\mathbf{v} - \mathbf{v}_i)$, where $\rho$ is the basis function (derivative reconstruction filter) [S3]. For voxels outside the support of $\rho$, zero weights are assigned [S3]. When data uncertainties are modeled as uniform distributions, the distribution of each partial derivative is a box spline [S3].

### Backprojection via Shepard's Interpolation
Fits a continuous scale field $S(x)$ from a discrete set of LoG extrema $N=\{(x_i, t_i)\}$ using basis functions [S4, unofficial]. The scale field is $S(x) = \sum_{x_i \in N} \Theta(||x - x_i||) t_i$ [S4, unofficial]. The basis function $\Theta(d)$ uses the fourth-degree Wendland polynomial: $\Theta(d) = [1 - d/h]_0^1)^4 (4d/h + 1)$ [S4, unofficial]. Parameter $h$ controls local support, typically $h = k \cdot t_i$ with $k=1$ [S4, unofficial]. An alternative blending uses the maximum operator $S(x) = \max_{x_i \in N} \{\Theta(||x - x_i||) t_i\}$ to prioritize larger features [S4, unofficial]. A pitfall is "bleeding," where small features inherit large scales, handled by a visibility test [S4, unofficial].

### Kriging Interpolation
Constructs a regular grid data model from limited discrete 3D borehole data [S5]. Steps: (1) Set grid range and size. (2) Calculate estimated point coordinates $p_0$. (3) Select reference points $p_i$ via search strategy. (4) Calculate coefficients $K, \lambda, D$ from the variation function. (5) Solve equation set using LU method to get weights $\lambda_i$. (6) Calculate property value $R'(p_0) = \sum_{i=1}^n \lambda_i R(p_i)$ and round to $R$ [S5]. Repeat for all grid points [S5].

### Zero Set Initialization of Isosurfaces
Initializes closest points for isosurfaces using local gradient estimates instead of trilinear interpolation [S6]. Examines each volume cell; if it contains the isosurface, initializes distance samples at corners with closest point info [S6]. Gradient $\nabla f(s)$ is estimated using central differencing [S6]. The closest point $s'$ is computed by solving $f(s') = f_0$ using linear approximation $f(s') = f(s) + \nabla f(s) \cdot (s' - s)$ [S6]. Inaccurate for high curvature regions; can be improved by Newton iterations [S6].

### Constant Filter Radius Mapping
Maps curvilinear grid data to a rectangular grid using a user-set constant filter radius [S7, unofficial]. Pitfall: Violates Nyquist criteria when input data has a wide range of scales and spatial densities; a single radius cannot adequately sample all parts of the input grid [S7, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, but with a high constant factor due to the 4x4x4 neighborhood access (64 values) and multiple interpolation steps. Gradient computation requires additional passes or precomputed vertex gradients.
- **Memory Complexity:** $O(N)$ for the grid storage. No additional memory per query.
- **Performance:** Slower than trilinear interpolation due to the larger stencil and cubic basis evaluations. Faster than unstructured spline methods.
- **Strengths:** $C^1$ continuity ensures smooth normals and stable ray marching. Better visual quality than linear interpolation.
- **Limitations:** High computational cost. Boundary handling requires careful clamping or extrapolation. Requires a dense grid.
- **When to use:** High-quality volume rendering, isosurface extraction requiring smooth normals, scientific visualization where derivative accuracy is critical.
- **When not to use:** Real-time applications with strict performance constraints, sparse data, or when sharp discontinuities are preferred (use nearest neighbor).

## Background
Originated in the context of isosurfacing and volume rendering, specifically in libraries like CGAL which provide `Interpolated_discrete_values_3` and `Interpolated_discrete_gradients_3` for contouring discrete data [S2]. The method is based on tri-cubic spline interpolation, a standard technique in computer graphics for smooth field reconstruction [S1, unofficial]. It is used in dual contouring and marching cubes algorithms to ensure geometric continuity [S2].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S3] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
[S4] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
[S5] Volume Rendering of 3D Borehole Data Based on GPU, https://doi.org/10.2991/icsem.2013.181, fetched 2026-10-26
[S6] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409942/?q=volume data compression rendering, fetched 2026-10-28
[S7] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
