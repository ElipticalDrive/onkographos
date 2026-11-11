# B-spline Field Interpolation

Also called: B-spline deformation field, Cubic B-spline field

## Overview
B-spline field interpolation is a method for evaluating a continuous function $u(\mathbf{p})$ at arbitrary query coordinates $\mathbf{p}$ using a grid of control points and cubic B-spline basis functions [S1, unofficial]. It builds on [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md) by using cubic polynomials that provide $C^2$ continuity and local support over a $4 \times 4 \times 4$ neighborhood [S1, unofficial]. Unlike [Voxel Data Interpolation and Sampling](12781-voxel-data-interpolation-and-sampling.md) which uses trilinear interpolation, this method produces smoother fields but requires evaluating 64 basis function products per query [S1, unofficial]. It is distinct from [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md) in its explicit use of parametric basis functions rather than simple corner averaging [S1, unofficial].

## How it works
**Inputs:**
- Query point $\mathbf{p} = (x, y, z)$ [S1, unofficial].
- Grid origin $\mathbf{o} = (x_0, y_0, z_0)$ [S1, unofficial].
- Grid spacing $\mathbf{s} = (dx, dy, dz)$ [S1, unofficial].
- Grid of control point weights $w_{i,j,k}$ [S1, unofficial].

**Outputs:**
- Interpolated scalar value $u$ at $\mathbf{p}$ [S1, unofficial].

**Data Structures:**
- The field is defined by a 3D grid of control points. Each control point has an integer index $(i_x, i_y, i_z)$ and a scalar weight $w_{i_x, i_y, i_z}$ [S1, unofficial].
- For vector fields (e.g., displacement), three independent scalar fields are used, one for each component $(u_x, u_y, u_z)$ [S1, unofficial].
- Minimum grid size is $4 \times 4 \times 4$ control points (64 weights) for a valid 3D field [S1, unofficial].

**Steps:**
1.  **Calculate Support Indices:** Determine the base indices $(i_x, i_y, i_z)$ for the $4 \times 4 \times 4$ support grid surrounding the query point [S1, unofficial]:
    $$
    i_x = \left\lfloor \frac{x - x_0}{dx} - 1 \right\rfloor, \quad
    i_y = \left\lfloor \frac{y - y_0}{dy} - 1 \right\rfloor, \quad
    i_z = \left\lfloor \frac{z - z_0}{dz} - 1 \right\rfloor
    $$
    [S1, unofficial]
2.  **Calculate Normalized Coordinates:** Compute the local coordinates $(\tilde{x}, \tilde{y}, \tilde{z})$ within the support region, where $0 \le \tilde{x}, \tilde{y}, \tilde{z} < 1$ [S1, unofficial]:
    $$
    \tilde{x} = \frac{x - x_0}{dx} - 1 - i_x, \quad
    \tilde{y} = \frac{y - y_0}{dy} - 1 - i_y, \quad
    \tilde{z} = \frac{z - z_0}{dz} - 1 - i_z
    $$
    [S1, unofficial]
3.  **Evaluate Basis Functions:** Compute the cubic B-spline basis functions $B_0, B_1, B_2, B_3$ for each dimension [S1, unofficial]:
    $$
    B_0(\tilde{x}) = \frac{(1 - \tilde{x})^3}{6}
    $$
    $$
    B_1(\tilde{x}) = \frac{3\tilde{x}^3 - 6\tilde{x}^2 + 4}{6}
    $$
    $$
    B_2(\tilde{x}) = \frac{-3\tilde{x}^3 + 3\tilde{x}^2 + 3\tilde{x} + 1}{6}
    $$
    $$
    B_3(\tilde{x}) = \frac{\tilde{x}^3}{6}
    $$
    [S1, unofficial]
4.  **Interpolate:** Sum the weighted contributions of the 64 control points in the support region [S1, unofficial]:
    $$
    u = \sum_{l=0}^3 \sum_{m=0}^3 \sum_{n=0}^3 B_l(\tilde{x}) B_m(\tilde{y}) B_n(\tilde{z}) w_{i_x+l, i_y+m, i_z+n}
    $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_b_spline_basis(t):
    """
    Evaluates the 4 cubic B-spline basis functions at normalized coordinate t.
    t must be in [0, 1).
    Returns list [B0, B1, B2, B3].
    """
    t2 = t * t
    t3 = t2 * t
    b0 = (1.0 - t) ** 3 / 6.0
    b1 = (3.0 * t3 - 6.0 * t2 + 4.0) / 6.0
    b2 = (-3.0 * t3 + 3.0 * t2 + 3.0 * t + 1.0) / 6.0
    b3 = t3 / 6.0
    return [b0, b1, b2, b3]

def interpolate_b_spline_field(x, y, z, origin, spacing, weights):
    """
    Interpolates value at (x, y, z) using B-spline field.
    
    Args:
        x, y, z: Query coordinates.
        origin: Tuple (x0, y0, z0).
        spacing: Tuple (dx, dy, dz).
        weights: 3D array or dictionary of control point weights w[i,j,k].
                 Must cover the support region.
    
    Returns:
        Interpolated scalar value u.
    """
    x0, y0, z0 = origin
    dx, dy, dz = spacing
    
    # Step 1: Calculate support indices
    ix = int((x - x0) / dx - 1)
    iy = int((y - y0) / dy - 1)
    iz = int((z - z0) / dz - 1)
    
    # Step 2: Calculate normalized coordinates
    x_tilde = (x - x0) / dx - 1 - ix
    y_tilde = (y - y0) / dy - 1 - iy
    z_tilde = (z - z0) / dz - 1 - iz
    
    # Step 3: Evaluate basis functions
    bx = cubic_b_spline_basis(x_tilde)
    by = cubic_b_spline_basis(y_tilde)
    bz = cubic_b_spline_basis(z_tilde)
    
    # Step 4: Interpolate
    u = 0.0
    for l in range(4):
        for m in range(4):
            for n in range(4):
                # Access weight w[ix+l, iy+m, iz+n]
                # Note: Implementation must handle boundary checks if indices are out of bounds
                w = weights[ix + l, iy + m, iz + n]
                u += bx[l] * by[m] * bz[n] * w
                
    return u
```

**Parameters:**
- **Grid Origin $(x_0, y_0, z_0)$**: The coordinate of the first control point in the grid [S1, unofficial].
- **Grid Spacing $(dx, dy, dz)$**: The distance between adjacent control points along each axis [S1, unofficial].
- **Control Point Weight $(w_{i_x, i_y, i_z})$**: The scalar value associated with the control point at index $(i_x, i_y, i_z)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Non-Interpolatory Nature**: The field value at a control point (where $\tilde{x}=0$) is not equal to the weight of that control point due to blending with neighbors [S1, unofficial]. For example, if a single control point has weight 1 and all others 0, the field value at that point is $2/3$ [S1, unofficial].
- **Boundary Handling**: The basis functions do not inherently define values outside the grid domain. Explicit boundary handling (e.g., clamping, periodic, or constant) is required for query points outside the defined control point grid [S1, Parent, unofficial].
- **Minimum Grid Size**: A minimum of $4 \times 4 \times 4$ control points is required for a valid 3D field [S1, unofficial].

## Subtopics

### Time-varying B-spline Deformation Field
A time-varying displacement field $u(\mathbf{x}, t)$ parametrized by cubic B-splines on a fixed 3D grid, allowing arbitrary continuous position and time queries [S1, unofficial]. It sets itself apart by generating time-varying weights $w(t)$ via a Multi-Layer Perceptron (MLP) $\phi(t) \to w(t)$, ensuring implicit time continuity and efficiency without discretizing time into timesteps [S1, unofficial]. Choose this when representing dynamic deformations where smooth temporal evolution is required [S1, unofficial]. The grid of control points remains fixed in space, while the weights vary with time [S1, unofficial]. Each grid point has trainable weights for each of the 3 displacement components ($u_x, u_y, u_z$) [S1, unofficial]. The interpolation formula is identical to the static case, but $w$ is replaced by $w(t)$ [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per query, as it always evaluates exactly 64 basis function products and weight multiplications [S1, unofficial].
- **Memory Complexity**: $O(N_x N_y N_z)$ for the control point grid, where $N_x, N_y, N_z$ are the number of control points in each dimension [S1, unofficial]. Minimum memory is 64 weights for a scalar field, or 192 weights for a vector field [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Expressive power increases with grid refinement (more control points) [S1, unofficial].
- **Strengths**: $C^2$ continuity, local support, smooth interpolation, efficient evaluation [S1, unofficial].
- **Limitations**: Not interpolatory (field values at control points differ from weights), requires boundary handling [S1, unofficial].
- **When to use**: When smooth, continuous fields are needed, such as in deformation fields for medical imaging or volume rendering [S1, unofficial].
- **When not to use**: When exact interpolation of control point values is required, or when memory is extremely constrained and a coarser, less smooth method is acceptable [S1, unofficial].

## Background
B-spline field interpolation was introduced in the context of neural rendering for dynamic tomography, specifically for representing time-varying deformation fields [S1, unofficial]. It builds on the cubic B-spline method by Rueckert et al. [S1, unofficial]. The method was modified to account for temporal variations by making the weights time-varying, generated by an MLP [S1, unofficial]. It is used in X-ray attenuation tomography to represent continuous density or displacement fields, contrasting with classical voxel-based representations [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
