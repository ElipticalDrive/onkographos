# Interpolated Discrete Values & Gradients

Also called: Cubic Spline Interpolation, CubicSpline, Discrete Field Interpolation, Interpolated Grid Fields, Tri-Cubic Interpolation

## Overview
Interpolated Discrete Values & Gradients computes a continuous scalar field and its spatial gradient from discrete data sampled at grid vertices using tri-cubic interpolation [S1, unofficial][S2]. It solves the problem of mapping continuous world-space coordinates to discrete memory indices while providing smooth derivatives essential for isosurfacing and ray marching [S1, unofficial][S2]. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md), which produces a piecewise-constant field with sharp discontinuities, this method blends adjacent voxels to ensure $C^0$ continuity of the value and $C^0$ continuity of the gradient (smooth first derivatives) [S1, unofficial][S2]. It differs from [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) by deriving gradients analytically from the interpolation basis functions rather than approximating them via finite differences on the raw discrete values [S2].

## How it works
**Inputs:**
- A query point $p = (p_x, p_y, p_z)$ in continuous space.
- A dense 3D grid of discrete scalar values $V$ stored at integer voxel coordinates. The grid is defined by a bounding box and voxel dimensions (spacing) [S1, unofficial][S2].
- Optional: Pre-computed discrete gradients at grid vertices, or a method to compute them from values [S2].

**Outputs:**
- A scalar value $f(p)$ interpolated at $p$.
- A gradient vector $\nabla f(p) = (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})$ at $p$.

**Data Structures:**
- **Grid:** A 3D array of size $D_x \times D_y \times D_z$. Each element stores a scalar value (e.g., density) [S2].
- **Voxel Coordinates:** Integer indices $(v_x, v_y, v_z)$ identifying the voxel containing $p$ [S1, unofficial].
- **Local Offset:** A fractional vector $t = (t_x, t_y, t_z)$ where each component is in $[0, 1)$, representing the position of $p$ within the voxel [S1, unofficial].

**Algorithm Steps:**

1.  **Coordinate Mapping:**
    Compute the integer voxel coordinates containing $p$:
    $$v_i[axis] = \lfloor \frac{p[axis] - bounds[0][axis]}{voxel\_dim[axis]} \rfloor$$
    where $bounds[0]$ is the minimum corner of the grid and $voxel\_dim$ is the spacing per axis [S1, unofficial].

2.  **Local Offset Calculation:**
    Compute the minimum corner of the voxel:
    $$v_{min} = bounds[0] + v_i \times voxel\_dim$$
    Compute the normalized offset $t$ in $[0, 1]$:
    $$t = \frac{p - v_{min}}{voxel\_dim}$$
    [S1, unofficial]

3.  **Neighborhood Selection:**
    Tri-cubic interpolation requires a $4 \times 4 \times 4$ neighborhood of voxels. The indices for the neighborhood along each axis $k \in \{x, y, z\}$ are $v_i[k]-1, v_i[k], v_i[k]+1, v_i[k]+2$ [S1, unofficial].
    *Edge Case:* If any index falls outside the grid boundaries, boundary conditions (e.g., clamping, reflection, or periodic) must be applied [G].

4.  **1D Cubic Interpolation (Hermite Basis):**
    The core interpolation function $CINT\_CR(val_0, val_1, val_2, val_3, t)$ performs cubic interpolation using the values at indices $-1, 0, 1, 2$ relative to the voxel corner, with parameter $t \in [0, 1]$ [S1, unofficial].
    
    The basis functions for cubic Hermite interpolation are defined as [S1, unofficial]:
    $$t_2 = t^2, \quad t_3 = t^2 \cdot t$$
    $$h_{00} = 2t_3 - 3t_2 + 1$$
    $$h_{10} = t_3 - 2t_2 + t$$
    $$h_{01} = -2t_3 + 3t_2$$
    $$h_{11} = t_3 - t_2$$
    
    Given two points $p_k, p_{k+1}$ and their derivatives $m_k, m_{k+1}$ over interval $\Delta = x_{k+1} - x_k$, the interpolated value is [S1, unofficial]:
    $$Result = h_{00} p_k + h_{10} \Delta m_k + h_{01} p_{k+1} + h_{11} \Delta m_{k+1}$$
    
    *Note:* In the standard tri-cubic volume rendering implementation described in [S1, unofficial], the derivatives $m$ are often approximated or assumed to be zero if not explicitly provided, reducing the basis to a standard cubic spline or Catmull-Rom variant. If derivatives are not available, the term involving $m$ is omitted or set to zero. The source [S1, unofficial] uses `CINT_CR` with 4 values, implying a Catmull-Rom spline where derivatives are estimated from the 4 surrounding points ($m_k = \frac{1}{2}(val_1 - val_{-1})$).

5.  **Tri-Cubic Interpolation Sequence:**
    Perform interpolation along the Z-axis for each $(i, j)$ in the $4 \times 4$ XY plane of the neighborhood [S1, unofficial]:
    $$z\_interp[j+1][i+1] = CINT\_CR(t_z, V[v_x+i-1][v_y+j-1][v_z-1], V[v_x+i-1][v_y+j-1][v_z], V[v_x+i-1][v_y+j-1][v_z+1], V[v_x+i-1][v_y+j-1][v_z+2])$$
    where $i, j \in \{-1, 0, 1, 2\}$.

    Interpolate the resulting Z-slices along the Y-axis [S1, unofficial]:
    $$y\_interp[i] = CINT\_CR(t_y, z\_interp[0][i], z\_interp[1][i], z\_interp[2][i], z\_interp[3][i])$$
    where $i \in \{0, 1, 2, 3\}$ (corresponding to $v_x-1$ to $v_x+2$).

    Interpolate the resulting Y-lines along the X-axis to get the final scalar value [S1, unofficial]:
    $$f(p) = CINT\_CR(t_x, y\_interp[0], y\_interp[1], y\_interp[2], y\_interp[3])$$

6.  **Gradient Computation:**
    The gradient $\nabla f(p)$ can be computed in two ways [S2]:
    - **Analytically:** Differentiate the basis functions $h_{00}, h_{10}, h_{01}, h_{11}$ with respect to $t$, then chain-rule through the coordinate transformation. This requires the discrete derivatives $m$ at the grid vertices.
    - **Discrete Approximation:** If analytical gradients are not stored, compute them from the discrete values using finite differences (e.g., central differences) before interpolation, or use a specialized gradient interpolation class [S2].
    
    In the CGAL implementation [S2], `Interpolated_discrete_gradients_3` computes the gradient field from the discrete values, either analytically if vertex gradients are known, or by calling `compute_discrete_gradients(values)` which likely uses finite differences.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_hermite_basis(t):
    """
    Computes the Hermite basis functions for a single interval.
    Assumes interval length delta = 1 for normalized t.
    If derivatives m are not provided, this reduces to Catmull-Rom logic 
    where m is estimated from neighbors. 
    Here we implement the standard Catmull-Rom spline using 4 points 
    v_{-1}, v_0, v_1, v_2 and parameter t in [0, 1].
    """
    t2 = t * t
    t3 = t2 * t
    
    # Catmull-Rom basis functions derived from Hermite with m = 0.5*(v1 - v_{-1})
    # h00 = 2t^3 - 3t^2 + 1
    # h10 = t^3 - 2t^2 + t  (coefficient for v0)
    # h01 = -2t^3 + 3t^2    (coefficient for v1)
    # h11 = t^3 - t^2       (coefficient for v2)
    # Note: Standard Catmull-Rom formula:
    # y = 0.5 * ((2*v0) + (-v_{-1} + v1)*t + (2*v_{-1} - 5*v0 + 4*v1 - v2)*t^2 + (-v_{-1} + 3*v0 - 3*v1 + v2)*t^3)
    # The basis functions below correspond to the Hermite form if derivatives are passed.
    # For volume rendering [S1, unofficial], CINT_CR takes 4 values.
    
    h00 = 2 * t3 - 3 * t2 + 1
    h10 = t3 - 2 * t2 + t
    h01 = -2 * t3 + 3 * t2
    h11 = t3 - t2
    
    return h00, h10, h01, h11

def catmull_rom(v_minus_1, v_0, v_1, v_2, t):
    """
    Performs 1D cubic interpolation using Catmull-Rom spline.
    """
    # Using the basis functions for Catmull-Rom directly
    # y = 0.5 * ( (2*v0) + (-v_{-1} + v1)*t + (2*v_{-1} - 5*v0 + 4*v1 - v2)*t^2 + (-v_{-1} + 3*v0 - 3*v1 + v2)*t^3 )
    t2 = t * t
    t3 = t2 * t
    
    term1 = 2 * v_0
    term2 = (-v_minus_1 + v_1) * t
    term3 = (2 * v_minus_1 - 5 * v_0 + 4 * v_1 - v_2) * t2
    term4 = (-v_minus_1 + 3 * v_0 - 3 * v_1 + v_2) * t3
    
    return 0.5 * (term1 + term2 + term3 + term4)

def interpolate_tricubic(grid, p, bounds, voxel_dim):
    """
    Interpolates a scalar value at point p using tri-cubic interpolation.
    """
    # 1. Compute voxel coordinates
    # v_i = floor((p - bounds[0]) / voxel_dim)
    v_i = [int((p[d] - bounds[0][d]) / voxel_dim[d]) for d in range(3)]
    
    # 2. Compute local offset t
    v_min = [bounds[0][d] + v_i[d] * voxel_dim[d] for d in range(3)]
    t = [(p[d] - v_min[d]) / voxel_dim[d] for d in range(3)]
    
    # 3. Tri-cubic interpolation
    # Interpolate along Z for each (i, j) in 4x4 neighborhood
    # i, j range from -1 to 2
    z_interp = [[0.0 for _ in range(4)] for _ in range(4)]
    
    for i in range(-1, 3):
        for j in range(-1, 3):
            # Get 4 Z-values: v_z-1, v_z, v_z+1, v_z+2
            # Indices: v_i[2]-1, v_i[2], v_i[2]+1, v_i[2]+2
            idx_z = [v_i[2] - 1, v_i[2], v_i[2] + 1, v_i[2] + 2]
            
            # Fetch values with boundary handling [G]
            vals = []
            for k in idx_z:
                vals.append(grid[v_i[0] + i][v_i[1] + j][k])
            
            z_interp[j + 1][i + 1] = catmull_rom(vals[0], vals[1], vals[2], vals[3], t[2])
            
    # Interpolate along Y
    y_interp = [0.0 for _ in range(4)]
    for i in range(4):
        # z_interp rows 0..3 correspond to j=-1..2
        vals = [z_interp[j][i] for j in range(4)]
        y_interp[i] = catmull_rom(vals[0], vals[1], vals[2], vals[3], t[1])
        
    # Interpolate along X
    vals = y_interp
    result = catmull_rom(vals[0], vals[1], vals[2], vals[3], t[0])
    
    return result

def compute_gradient_analytical(grid, p, bounds, voxel_dim):
    """
    Computes gradient by differentiating the interpolation basis.
    This requires implementing the derivative of the Catmull-Rom spline.
    """
    # Derivative of Catmull-Rom:
    # dy/dt = 0.5 * ( (-v_{-1} + v1) + 2*(2*v_{-1} - 5*v0 + 4*v1 - v2)*t + 3*(-v_{-1} + 3*v0 - 3*v1 + v2)*t^2 )
    # Then chain rule: df/dx = (df/dt) * (dt/dx) = (df/dt) / voxel_dim
    
    # Implementation omitted for brevity, follows same structure as interpolate_tricubic
    # but uses derivative basis functions.
    pass
```

## Subtopics

### Uncertain Gradient Field Interpolation
Reconstructs the gradient field from an uncertain scalar field, resulting in a trivariate random vector $Y$ representing the uncertainties of partial derivatives along $x, y, z$ [S3]. It differs from standard gradient interpolation by propagating the statistical uncertainty of the input scalar values through the interpolation weights [S3]. The gradient is computed as a linear combination of random variables $X_i$ at neighboring voxels, weighted by a basis function $\rho$ (typically a combination of linear interpolation and central difference) [S3]. If data uncertainties are modeled as uniform distributions, the resulting gradient distribution is a trivariate box spline [S3].

### Backprojection via Shepard's Interpolation
A technique to fit a continuous scale field $S(x)$ from a discrete set of LoG (Laplacian of Gaussian) extrema $N=\{(x_i, t_i)\}$ [S4, unofficial]. It uses a fourth-degree Wendland polynomial basis function with compact support to interpolate the representative scales $t_i$ [S4, unofficial]. Unlike standard tri-cubic interpolation which blends all neighbors, this method can use a maximum operator ($S(x) = \max \Theta(||x-x_i||) t_i$) to prioritize larger features and avoid smoothing away scale variation [S4, unofficial]. A key pitfall is "bleeding," where a small feature inherits the scale of a large nearby feature, which is mitigated by a visibility test [S4, unofficial].

### Kriging Interpolation
An interpolation algorithm used to construct a regular grid data model from limited and discrete 3D borehole data [S5]. It differs from deterministic cubic interpolation by using a variogram-based statistical model to determine optimal weights $\lambda_i$ for the estimated point value $R'(p_0) = \sum \lambda_i R(p_i)$ [S5]. It solves the problem of irregularly spaced data by selecting reference points based on a search strategy and solving a linear equation set via LU decomposition [S5].

### Zero Set Initialization of Isosurfaces
Initializes closest points for isosurfaces defined on regularly sampled scalar fields using local gradient estimates instead of trilinear interpolation [S6]. It differs from standard value interpolation by focusing on initializing signed-distance volumes. It estimates the scalar field gradient $\nabla f(s)$ at sample $s$ using central differencing and computes the closest point on the isosurface via linear approximation [S6]. It is inaccurate for high-curvature regions but computationally efficient [S6].

### Constant Filter Radius Mapping
Maps curvilinear grid data to a rectangular grid using a user-set constant filter radius for spatial interpolation [S7, unofficial]. It differs from adaptive methods by using a fixed radius, which can violate the Nyquist criteria if the input data has a wide range of scales and spatial densities [S7, unofficial]. A single radius cannot adequately sample all parts of the input grid, leading to loss of data accuracy in either dense or sparse regions [S7, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query point, as it involves a fixed $4 \times 4 \times 4$ neighborhood lookup and a constant number of arithmetic operations.
- **Memory Complexity:** $O(1)$ auxiliary memory per query, plus $O(N)$ for the grid storage where $N$ is the number of voxels [S1, unofficial][S2].
- **Performance:** Tri-cubic interpolation is more computationally expensive than trilinear or nearest-neighbor interpolation due to the larger neighborhood and cubic basis calculations [S1, unofficial]. However, it provides significantly higher visual quality and smoother gradients [S1, unofficial].
- **Strengths:** Provides $C^0$ continuous values and smooth gradients, which are essential for high-quality isosurfacing (e.g., Dual Contouring) and ray marching [S1, unofficial][S2].
- **Limitations:** Requires a $4 \times 4 \times 4$ neighborhood, which increases memory bandwidth usage compared to trilinear interpolation [S1, unofficial]. Boundary conditions must be carefully handled to avoid artifacts [G].

## Background
The method originates from the use of cubic spline interpolation in volume rendering and isosurfacing. It is implemented in libraries such as CGAL's Isosurfacing module (`Interpolated_discrete_values_3` and `Interpolated_discrete_gradients_3`) for generating isosurfaces from discrete data [S2]. The specific tri-cubic implementation with a $4 \times 4 \times 4$ neighborhood is described in CUDA-based volume rendering experiments [S1, unofficial]. The concept of interpolating gradients from discrete values is also discussed in the context of uncertain scalar fields and size-based transfer functions [S3][S4, unofficial].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S2] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S3] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
[S4] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
[S5] Volume Rendering of 3D Borehole Data Based on GPU, https://doi.org/10.2991/icsem.2013.181, fetched 2026-10-26
[S6] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409942/?q=volume data compression rendering, fetched 2026-10-28
[S7] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
