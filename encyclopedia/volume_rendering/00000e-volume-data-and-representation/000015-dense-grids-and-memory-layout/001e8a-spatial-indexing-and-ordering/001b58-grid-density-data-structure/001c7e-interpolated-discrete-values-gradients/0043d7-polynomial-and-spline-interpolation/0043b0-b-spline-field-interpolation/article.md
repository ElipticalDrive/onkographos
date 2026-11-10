# B-spline Field Interpolation
Also called: B-spline deformation field, Cubic B-spline field

## Overview
B-spline Field Interpolation is a method for evaluating continuous scalar or vector fields from a discrete grid of control points using cubic B-spline basis functions [S1, unofficial]. It solves the problem of querying a smooth, differentiable field at arbitrary continuous coordinates while maintaining local support, meaning only a fixed, small number of grid points influence any single query [S1, unofficial]. It is distinguished from its parent [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md) by its specific use of cubic B-splines, which provide $C^2$ continuity and a fixed 4x4x4 support kernel in 3D, and from its sibling [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) by its higher-order continuity and larger support footprint [S1, unofficial].

## How it works
**Inputs:**
*   Query point $\mathbf{p} = (x, y, z)$.
*   Grid origin $\mathbf{x}_0 = (x_0, y_0, z_0)$.
*   Grid spacing $\mathbf{dx} = (dx, dy, dz)$.
*   Control point weights $w_{i,j,k}$ for all relevant indices.

**Outputs:**
*   Interpolated value $u(\mathbf{p})$.

**Data Structures:**
*   **Control Point Grid:** A 3D array of weights. For a scalar field, each element is a scalar. For a vector field (e.g., displacement), each grid point stores three weights $(w_x, w_y, w_z)$.
*   **Support Region:** Defined by the 4x4x4 block of control points surrounding the query point.

**Steps:**
1.  **Calculate Integer Indices:** Determine the base indices $(i_x, i_y, i_z)$ of the support grid for the query point $\mathbf{p}$:
    $$ i_x = \left\lfloor \frac{x - x_0}{dx} - 1 \right\rfloor $$
    $$ i_y = \left\lfloor \frac{y - y_0}{dy} - 1 \right\rfloor $$
    $$ i_z = \left\lfloor \frac{z - z_0}{dz} - 1 \right\rfloor $$
    The support region spans indices $[i_x, i_x+3]$, $[i_y, i_y+3]$, and $[i_z, i_z+3]$ [S1, unofficial].

2.  **Calculate Normalized Coordinates:** Compute the local coordinates $(\tilde{x}, \tilde{y}, \tilde{z})$ within the support region:
    $$ \tilde{x} = \frac{x - x_0}{dx} - 1 - i_x $$
    $$ \tilde{y} = \frac{y - y_0}{dy} - 1 - i_y $$
    $$ \tilde{z} = \frac{z - z_0}{dz} - 1 - i_z $$
    These values lie in the range $[0, 1)$ [S1, unofficial].

3.  **Evaluate Basis Functions:** Compute the cubic B-spline basis functions $B_0, B_1, B_2, B_3$ for each dimension using $\tilde{x}, \tilde{y}, \tilde{z}$:
    $$ B_0(\tilde{t}) = \frac{(1 - \tilde{t})^3}{6} $$
    $$ B_1(\tilde{t}) = \frac{3\tilde{t}^3 - 6\tilde{t}^2 + 4}{6} $$
    $$ B_2(\tilde{t}) = \frac{-3\tilde{t}^3 + 3\tilde{t}^2 + 3\tilde{t} + 1}{6} $$
    $$ B_3(\tilde{t}) = \frac{\tilde{t}^3}{6} $$
    where $\tilde{t}$ is $\tilde{x}$, $\tilde{y}$, or $\tilde{z}$ [S1, unofficial].

4.  **Compute Interpolated Value:** Sum the weighted contributions of the 64 control points in the support region:
    $$ u(\mathbf{p}) = \sum_{l=0}^3 \sum_{m=0}^3 \sum_{n=0}^3 B_l(\tilde{x}) B_m(\tilde{y}) B_n(\tilde{z}) w_{i_x+l, i_y+m, i_z+n} $$
    For vector fields, this is computed independently for each component [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_b_spline_basis(t):
    """
    Evaluates the 4 cubic B-spline basis functions for a normalized coordinate t in [0, 1).
    Returns a list [B0, B1, B2, B3].
    """
    t2 = t * t
    t3 = t2 * t
    b0 = (1.0 - t) ** 3 / 6.0
    b1 = (3.0 * t3 - 6.0 * t2 + 4.0) / 6.0
    b2 = (-3.0 * t3 + 3.0 * t2 + 3.0 * t + 1.0) / 6.0
    b3 = t3 / 6.0
    return [b0, b1, b2, b3]

def interpolate_b_spline_field(x, y, z, x0, y0, z0, dx, dy, dz, weights):
    """
    Interpolates a value from a 3D B-spline field.
    
    Args:
        x, y, z: Query point coordinates.
        x0, y0, z0: Grid origin.
        dx, dy, dz: Grid spacing.
        weights: 3D array of control point weights.
        
    Returns:
        Interpolated scalar value.
    """
    # Step 1: Calculate integer indices
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
    
    # Step 4: Compute interpolated value
    value = 0.0
    for l in range(4):
        for m in range(4):
            for n in range(4):
                # Access weight at index (ix+l, iy+m, iz+n)
                # Note: In practice, boundary checks are required if the query point
                # is outside the defined grid range. [G]
                w = weights[ix + l, iy + m, iz + n]
                value += bx[l] * by[m] * bz[n] * w
                
    return value
```

**Parameters:**
*   **Grid Origin:** $[x_0, y_0, z_0]$, the coordinate of the first control point [S1, unofficial].
*   **Grid Spacing:** $dx, dy, dz$, the distance between adjacent control points [S1, unofficial].
*   **Control Point Weight:** $w_{i_x, i_y, i_z}$, the scalar value associated with a specific grid index [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Interpolation Property:** The field value at a control point location ($\tilde{x}=0$) is not equal to the weight of that control point due to the blending of neighboring weights. For example, with a single non-zero weight of 1, the field value at that point is $2/3$ [S1, unofficial].
*   **Minimum Grid Size:** A 3D B-spline field requires a minimum of $4 \times 4 \times 4 = 64$ control points to define a valid support region [S1, unofficial].
*   **Boundary Conditions:** The sources do not specify behavior for query points outside the grid boundaries. Implementers must define clamping, extrapolation, or error handling [G].

## Subtopics

### Time-varying B-spline Deformation Field
A variant where the displacement field $\mathbf{u}(\mathbf{x}, t)$ is parametrized by cubic B-splines on a fixed 3D spatial grid, with weights varying over time [S1, unofficial]. It allows arbitrary continuous position and time queries [S1, unofficial]. The grid control points remain fixed in space, but the weights $w(t)$ are generated implicitly as a function of time $t$ using a Multi-Layer Perceptron (MLP) $\phi(t) \rightarrow w(t)$ [S1, unofficial]. This approach ensures implicit time continuity and is more efficient than discretizing time into timesteps [S1, unofficial]. For a 3D displacement field, the minimum degrees of freedom is $3 \times 64 = 192$ weights [S1, unofficial]. The interpolation formula remains the same, but $w$ becomes $w(t)$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, as it always involves evaluating 64 basis function products and summing them.
*   **Memory Complexity:** $O(N_x N_y N_z)$ for storing control point weights. For vector fields, memory usage is tripled [S1, unofficial].
*   **Strengths:** Provides $C^2$ continuous fields with local support, enabling smooth deformations and efficient querying. The fixed support size makes it predictable and cache-friendly [S1, unofficial].
*   **Limitations:** The non-interpolation property means control points do not pass through the weights exactly. Requires a minimum grid size of 4x4x4 [S1, unofficial].
*   **When to use:** Suitable for representing smooth, continuous fields such as deformation fields in medical imaging or volumetric data where smoothness and differentiability are required [S1, unofficial].
*   **When not to use:** When exact interpolation of control point values is required, or when the field has sharp discontinuities that B-splines may smooth out excessively [S1, unofficial].

## Background
B-spline field interpolation was adapted for neural rendering in the context of dynamic tomography, specifically for representing time-varying deformation fields [S1, unofficial]. The method builds on the cubic B-spline formulation by Rueckert et al. [19] (cited in [S1, unofficial]). It was modified to account for temporal variations by making the weights time-dependent, generated via an MLP, allowing for efficient and continuous representation of dynamic volumetric data [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
