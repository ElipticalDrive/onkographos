# B-spline Field Interpolation

Also called: B-spline deformation field, Cubic B-spline field

## Overview
B-spline field interpolation is a method for evaluating a continuous function from a discrete grid of control points using cubic B-splines, where the basis functions have local support [S1, unofficial]. It differs from nearest-neighbor or trilinear interpolation by providing $C^1$ continuity (continuous first derivatives) and smoother results, at the cost of a larger support region (64 control points in 3D) and higher computational overhead per query [S1, unofficial]. It is distinct from Piecewise Quadratic Hermite Interpolation, which uses fewer points but lower continuity, and Voxel Data Interpolation, which typically refers to lower-order or non-spline methods [S1, unofficial].

## How it works
**Inputs:**
*   Query point $\mathbf{p} = (x, y, z)$ in continuous space.
*   Grid origin $\mathbf{o} = (x_0, y_0, z_0)$.
*   Grid spacing $\mathbf{h} = (dx, dy, dz)$.
*   Control point weights $w_{i,j,k}$ for a 3D grid of indices $(i, j, k)$. For vector fields, three independent scalar weight grids are used (one per component) [S1, unofficial].

**Outputs:**
*   Interpolated scalar value $u(\mathbf{p})$ or vector $\mathbf{u}(\mathbf{p})$.

**Data Structures:**
*   The weights are stored in a 3D array or flattened 1D array. The minimum grid size for a valid 3D field is $4 \times 4 \times 4$ (64 points) [S1, unofficial].
*   No explicit derivative storage is required; gradients are computed analytically from the basis functions if needed, though the primary method focuses on value interpolation [S1, unofficial].

**Steps:**
1.  **Index Calculation**: Determine the integer indices $(i_x, i_y, i_z)$ of the "anchor" control point. The support region consists of the $4 \times 4 \times 4$ grid of points starting from these indices [S1, unofficial]:
    $$
    i_x = \left\lfloor \frac{x - x_0}{dx} - 1 \right\rfloor, \quad
    i_y = \left\lfloor \frac{y - y_0}{dy} - 1 \right\rfloor, \quad
    i_z = \left\lfloor \frac{z - z_0}{dz} - 1 \right\rfloor
    $$
    where $\lfloor \cdot \rfloor$ denotes the floor function [S1, unofficial].

2.  **Normalized Coordinate Calculation**: Compute the local coordinates $(\tilde{x}, \tilde{y}, \tilde{z})$ within the support cell. These coordinates lie in the range $[0, 1]$ [S1, unofficial]:
    $$
    \tilde{x} = \frac{x - x_0}{dx} - 1 - i_x, \quad
    \tilde{y} = \frac{y - y_0}{dy} - 1 - i_y, \quad
    \tilde{z} = \frac{z - z_0}{dz} - 1 - i_z
    $$

3.  **Basis Evaluation**: Evaluate the 1D cubic B-spline basis functions $B_0, B_1, B_2, B_3$ for each dimension. The basis functions are defined as [S1, unofficial]:
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
    Note that $\sum_{k=0}^3 B_k(\tilde{x}) = 1$ for all $\tilde{x} \in [0, 1]$ [S1, unofficial].

4.  **Weighted Summation**: Compute the interpolated value by summing the products of the basis weights and the control point weights over the $4 \times 4 \times 4$ support [S1, unofficial]:
    $$
    u(\mathbf{p}) = \sum_{l=0}^3 \sum_{m=0}^3 \sum_{n=0}^3 B_l(\tilde{x}) B_m(\tilde{y}) B_n(\tilde{z}) w_{i_x+l, i_y+m, i_z+n}
    $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_b_spline_3d(x, y, z, x0, y0, z0, dx, dy, dz, weights):
    """
    Interpolate a value at (x, y, z) using a 3D grid of B-spline control points.
    
    Args:
        x, y, z: Query coordinates.
        x0, y0, z0: Grid origin.
        dx, dy, dz: Grid spacing.
        weights: 3D array of control point weights.
        
    Returns:
        Interpolated scalar value.
    """
    # 1. Calculate integer indices
    ix = int((x - x0) / dx - 1)
    iy = int((y - y0) / dy - 1)
    iz = int((z - z0) / dz - 1)
    
    # 2. Calculate normalized coordinates
    tx = (x - x0) / dx - 1 - ix
    ty = (y - y0) / dy - 1 - iy
    tz = (z - z0) / dz - 1 - iz
    
    # 3. Evaluate basis functions
    def b0(t): return (1 - t)**3 / 6.0
    def b1(t): return (3*t**3 - 6*t**2 + 4) / 6.0
    def b2(t): return (-3*t**3 + 3*t**2 + 3*t + 1) / 6.0
    def b3(t): return t**3 / 6.0
    
    bx = [b0(tx), b1(tx), b2(tx), b3(tx)]
    by = [b0(ty), b1(ty), b2(ty), b3(ty)]
    bz = [b0(tz), b1(tz), b2(tz), b3(tz)]
    
    # 4. Weighted summation
    value = 0.0
    for l in range(4):
        for m in range(4):
            for n in range(4):
                # Access weight at (ix+l, iy+m, iz+n)
                # Note: In practice, bounds checking for grid edges is required [G]
                w = weights[ix + l, iy + m, iz + n]
                value += bx[l] * by[m] * bz[n] * w
                
    return value
```

**Parameters:**
*   Grid origin $[x_0, y_0, z_0]$: The coordinate of the first control point [S1, unofficial].
*   Grid spacing $dx, dy, dz$: Distance between adjacent control points [S1, unofficial].
*   Control point weight $w_{i,j,k}$: The scalar value associated with the control point at grid index $(i, j, k)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-interpolatory Property**: The value of the field at a control point (where $\tilde{x}=0, \tilde{y}=0, \tilde{z}=0$) is not equal to the weight of that control point. For example, if only one control point has weight 1 and others are 0, the field value at that point is $2/3$ due to blending with neighbors [S1, unofficial].
*   **Boundary Conditions**: The method assumes the query point falls within the valid range of the grid. Queries outside the grid bounds require extrapolation or clamping strategies not defined in the base method [G].
*   **Precision**: The basis functions are cubic polynomials. Standard floating-point precision is sufficient, but care should be taken with very small or very large grid spacings to avoid normalization errors [G].

## Subtopics

### Time-varying B-spline Deformation Field
A variant where the displacement field $\mathbf{u}(\mathbf{x}, t)$ is parametrized by cubic B-splines on a fixed 3D spatial grid, but the weights $w(t)$ vary with time $t$ [S1, unofficial]. Instead of storing discrete timesteps, the weights are generated implicitly as a function of time, typically via a Multi-Layer Perceptron (MLP) $\phi(t) \rightarrow w(t)$ [S1, unofficial]. This approach allows arbitrary continuous position and time queries, ensures implicit time continuity, and is more efficient than storing discrete weight sets for many timesteps [S1, unofficial]. The spatial interpolation remains identical to the static case, using the time-dependent weights $w(t)_{i,j,k}$ [S1, unofficial].

## Operating characteristics
*   **Complexity**: Time complexity is $O(1)$ per query, specifically requiring $4^3 = 64$ weight lookups and polynomial evaluations in 3D. Memory complexity is proportional to the number of control points ($N_x N_y N_z$).
*   **Performance**: The fixed support size makes it predictable, but 64 lookups are significantly more expensive than trilinear (8 lookups) or nearest-neighbor (1 lookup) [S1, unofficial].
*   **Strengths**: Provides smooth, $C^1$ continuous fields, which is critical for deformation fields in medical imaging and volume rendering to avoid artifacts. Local support ensures that changes to one control point only affect a local region [S1, unofficial].
*   **Limitations**: The non-interpolatory nature means the field does not pass exactly through the control points, which may be undesirable for certain data fitting tasks. The larger support size increases memory bandwidth usage [S1, unofficial].
*   **When to use**: Ideal for representing smooth deformation fields, density fields, or any continuous scalar/vector field where smoothness is prioritized over exact interpolation of control data [S1, unofficial].

## Background
The method is based on cubic B-splines, originally introduced by de Boor and others in the context of computer-aided geometric design. In the context of volume rendering and medical imaging, it was adapted for deformation fields by Rueckert et al. [S1, unofficial]. The specific implementation details, including the index calculation and basis function definitions provided here, are derived from a neural rendering framework for dynamic tomography [S1, unofficial]. The approach was chosen to allow continuous spatiotemporal querying of deformation fields without discretizing time, improving efficiency and continuity [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
