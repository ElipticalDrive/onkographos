# B-spline Field Interpolation

Also called: B-spline deformation field, Cubic B-spline field

## Overview
B-spline Field Interpolation is a method for function interpolation using cubic B-splines, defined by a grid of control points and weights, where shape functions have local influence [S1, unofficial]. It solves the problem of representing continuous scalar or vector fields from discrete control points, ensuring $C^2$ continuity and local support, which is particularly useful for deformation fields in neural rendering and tomography [S1, unofficial]. It sets itself apart from linear interpolation (which is only $C^0$) and global basis functions (like Fourier) by providing smooth, localized blending of control point weights [S1, unofficial].

## How it works
**Inputs:**
- Query point $\mathbf{p} = (x, y, z)$.
- Grid origin $\mathbf{p}_0 = (x_0, y_0, z_0)$.
- Grid spacing $\mathbf{s} = (dx, dy, dz)$.
- Control point weights $w_{i,j,k}$ for a 3D grid of indices $(i,j,k)$.

**Outputs:**
- Interpolated scalar value $u(\mathbf{p})$. For vector fields, this is repeated for each component.

**Data Structures:**
- The field is defined on a regular 3D grid.
- Minimum grid size for a valid 3D field is $4 \times 4 \times 4$ control points [S1, unofficial].
- For a vector field (e.g., 3D displacement), three independent sets of weights are stored, one for each component ($u_x, u_y, u_z$), totaling $3 \times 64 = 192$ degrees of freedom for the minimum grid [S1, unofficial].

**Steps:**
1.  **Calculate Integer Indices:** Determine the base indices of the $4 \times 4 \times 4$ support grid for the query point.
    $$ i_x = \left\lfloor \frac{x - x_0}{dx} - 1 \right\rfloor $$
    $$ i_y = \left\lfloor \frac{y - y_0}{dy} - 1 \right\rfloor $$
    $$ i_z = \left\lfloor \frac{z - z_0}{dz} - 1 \right\rfloor $$
    The support region spans indices $[i_x, i_x+3]$, $[i_y, i_y+3]$, and $[i_z, i_z+3]$ [S1, unofficial].

2.  **Calculate Normalized Coordinates:** Compute the local coordinates $\tilde{x}, \tilde{y}, \tilde{z}$ within the unit cube of the support region.
    $$ \tilde{x} = \frac{x - x_0}{dx} - 1 - i_x $$
    $$ \tilde{y} = \frac{y - y_0}{dy} - 1 - i_y $$
    $$ \tilde{z} = \frac{z - z_0}{dz} - 1 - i_z $$
    These values lie in the range $[0, 1]$ [S1, unofficial].

3.  **Evaluate Basis Functions:** Compute the cubic B-spline basis functions $B_0, B_1, B_2, B_3$ for each dimension using $\tilde{x}, \tilde{y}, \tilde{z}$ [S1, unofficial].

4.  **Compute Interpolated Value:** Sum the weighted contributions of the 64 control points in the support region.
    $$ u = \sum_{l=0}^3 \sum_{m=0}^3 \sum_{n=0}^3 B_l(\tilde{x}) B_m(\tilde{y}) B_n(\tilde{z}) w_{i_x+l, i_y+m, i_z+n} $$
    [S1, unofficial]

**Basis Functions:**
The cubic B-spline basis functions $B_k(\tilde{t})$ for $\tilde{t} \in [0, 1]$ are defined as [S1, unofficial]:
$$ B_0(\tilde{t}) = \frac{(1 - \tilde{t})^3}{6} $$
$$ B_1(\tilde{t}) = \frac{3\tilde{t}^3 - 6\tilde{t}^2 + 4}{6} $$
$$ B_2(\tilde{t}) = \frac{-3\tilde{t}^3 + 3\tilde{t}^2 + 3\tilde{t} + 1}{6} $$
$$ B_3(\tilde{t}) = \frac{\tilde{t}^3}{6} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_b_spline_basis(t):
    """
    Computes the 4 cubic B-spline basis values for a normalized coordinate t in [0, 1].
    """
    t2 = t * t
    t3 = t2 * t
    b0 = (1.0 - t) ** 3 / 6.0
    b1 = (3.0 * t3 - 6.0 * t2 + 4.0) / 6.0
    b2 = (-3.0 * t3 + 3.0 * t2 + 3.0 * t + 1.0) / 6.0
    b3 = t3 / 6.0
    return b0, b1, b2, b3

def b_spline_field_interpolation(x, y, z, x0, y0, z0, dx, dy, dz, weights):
    """
    Interpolates a value from a 3D B-spline field.
    
    Args:
        x, y, z: Query point coordinates.
        x0, y0, z0: Grid origin.
        dx, dy, dz: Grid spacing.
        weights: 3D array of control point weights, indexed as weights[ix, iy, iz].
                 Must be at least 4x4x4 in size.
    
    Returns:
        Interpolated scalar value u.
    """
    # 1. Calculate integer indices
    ix = int((x - x0) / dx - 1)
    iy = int((y - y0) / dy - 1)
    iz = int((z - z0) / dz - 1)
    
    # 2. Calculate normalized coordinates
    x_tilde = (x - x0) / dx - 1 - ix
    y_tilde = (y - y0) / dy - 1 - iy
    z_tilde = (z - z0) / dz - 1 - iz
    
    # 3. Evaluate basis functions
    bx = cubic_b_spline_basis(x_tilde)
    by = cubic_b_spline_basis(y_tilde)
    bz = cubic_b_spline_basis(z_tilde)
    
    # 4. Compute interpolated value
    u = 0.0
    for l in range(4):
        for m in range(4):
            for n in range(4):
                # Access weight at index (ix+l, iy+m, iz+n)
                # Note: In a real implementation, boundary checks or padding 
                # would be required if indices fall outside the grid bounds.
                w = weights[ix + l, iy + m, iz + n]
                u += bx[l] * by[m] * bz[n] * w
                
    return u
```

**Parameters:**
- Grid origin $[x_0, y_0, z_0]$: The spatial origin of the control point grid [S1, unofficial].
- Grid spacing $dx, dy, dz$: The distance between adjacent control points along each axis [S1, unofficial].
- Control point weight $w_{i_x, i_y, i_z}$: The scalar value associated with the control point at grid indices $(i_x, i_y, i_z)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- The field value at a control point location (where $\tilde{x}=0, \tilde{y}=0, \tilde{z}=0$) is **not** equal to the weight of that control point. Due to the blending of neighboring weights, the value is a weighted sum of the 64 surrounding weights. For example, if only one control point has weight 1 and all others are 0, the field value at that point is $2/3$, not 1 [S1, unofficial].
- The minimum number of control points for a 3D field is $4 \times 4 \times 4 = 64$. Fewer points do not define a valid field for this interpolation scheme [S1, unofficial].

## Subtopics

### Time-varying B-spline Deformation Field
A time-varying displacement field $u(\mathbf{x}, t)$ parametrized by cubic B-splines on a fixed 3D grid, allowing arbitrary continuous position and time queries [S1, unofficial]. It sets itself apart from static fields by making the weights time-dependent, typically generated by a Multi-Layer Perceptron (MLP) $\phi(t) \to w(t)$, which ensures implicit time continuity and efficiency without discretizing time into timesteps [S1, unofficial]. Choose this when representing dynamic deformations in neural rendering or tomography where smooth temporal evolution is required [S1, unofficial]. The grid of control points remains fixed in space, while the weights $w(t)$ vary with time [S1, unofficial]. The displacement component $u$ at position $(x,y,z)$ and time $t$ is calculated by interpolating the time-varying weights of the 64 closest grid points using the same cubic B-spline basis functions as the static case [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, as it always involves a fixed number of operations (64 weight lookups and basis evaluations) regardless of grid size [S1, unofficial].
- **Memory Complexity:** $O(N_x N_y N_z)$, where $N_x, N_y, N_z$ are the dimensions of the control point grid. For a vector field, memory usage is tripled [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Local support ensures that changes to a control point only affect a small region around it. $C^2$ continuity provides smooth deformations. Efficient for dynamic fields when combined with MLPs for weight generation [S1, unofficial].
- **Limitations:** The field value at control points does not match the control point weights, which can be counter-intuitive. Requires a minimum grid size of $4 \times 4 \times 4$ [S1, unofficial].
- **When to use:** Suitable for representing smooth, continuous deformation fields in volume rendering, particularly in dynamic tomography or medical image registration [S1, unofficial].
- **When not to use:** Not suitable for fields requiring exact interpolation at control points (i.e., $u(\mathbf{p}_i) = w_i$) without post-processing or correction [S1, unofficial].

## Background
The method is based on cubic B-splines as described by Rueckert et al. [19] (cited in [S1, unofficial]). It was modified in the context of neural rendering for dynamic tomography to account for temporal variations in deformation [S1, unofficial]. The approach was compared against other methods for handling time-varying fields, with the implicit time-dependent weight generation via MLPs yielding the best results [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
