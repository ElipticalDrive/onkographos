# B-spline Field Interpolation

Also called: B-spline deformation field, Cubic B-spline field

## Overview
B-spline Field Interpolation is a method for evaluating a continuous scalar or vector field from a discrete grid of control points using cubic B-spline basis functions [S1, unofficial]. It solves the problem of reconstructing a smooth, differentiable function from sparse samples while maintaining local support, ensuring that query complexity remains constant relative to the grid size [S1, unofficial]. It is distinguished from its parent [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md) by its specific use of cubic B-splines, which provide $C^2$ continuity and a fixed 4x4x4 support region in 3D, and from its sibling [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) by its higher-order continuity and cubic polynomial basis [S1, unofficial].

## How it works
**Inputs:**
*   Query point $\mathbf{p} = (x, y, z)$ [S1, unofficial].
*   Grid origin $\mathbf{p}_0 = (x_0, y_0, z_0)$ [S1, unofficial].
*   Grid spacing $\mathbf{s} = (dx, dy, dz)$ [S1, unofficial].
*   Weight grid $W$, where $W_{i,j,k}$ is the scalar weight associated with control point indices $(i, j, k)$ [S1, unofficial].

**Outputs:**
*   Interpolated value $u(\mathbf{p})$ [S1, unofficial].

**Data Structures:**
*   The weight grid $W$ is a 3D array. For a scalar field, each element stores one weight. For a vector field (e.g., displacement), three separate weight grids (or a single grid with 3 components per entry) are required, one for each spatial component ($u_x, u_y, u_z$) [S1, unofficial].
*   Minimum grid size is $4 \times 4 \times 4$ control points to define a valid field [S1, unofficial].

**Steps:**
1.  **Determine Support Indices:** Calculate the integer indices $(i_x, i_y, i_z)$ of the bottom-left corner of the $4 \times 4 \times 4$ support cube containing the query point [S1, unofficial]:
    $$
    i_x = \left\lfloor \frac{x - x_0}{dx} - 1 \right\rfloor, \quad
    i_y = \left\lfloor \frac{y - y_0}{dy} - 1 \right\rfloor, \quad
    i_z = \left\lfloor \frac{z - z_0}{dz} - 1 \right\rfloor
    $$
    [S1, unofficial]

2.  **Calculate Normalized Coordinates:** Compute the local coordinates $(\tilde{x}, \tilde{y}, \tilde{z})$ within the support region, normalized to the range $[0, 1]$ [S1, unofficial]:
    $$
    \tilde{x} = \frac{x - x_0}{dx} - 1 - i_x, \quad
    \tilde{y} = \frac{y - y_0}{dy} - 1 - i_y, \quad
    \tilde{z} = \frac{z - z_0}{dz} - 1 - i_z
    $$
    [S1, unofficial]

3.  **Evaluate Basis Functions:** Compute the cubic B-spline basis values for each dimension using the normalized coordinates [S1, unofficial]:
    $$
    \begin{aligned}
    B_0(\tilde{x}) &= \frac{(1 - \tilde{x})^3}{6} \\
    B_1(\tilde{x}) &= \frac{3\tilde{x}^3 - 6\tilde{x}^2 + 4}{6} \\
    B_2(\tilde{x}) &= \frac{-3\tilde{x}^3 + 3\tilde{x}^2 + 3\tilde{x} + 1}{6} \\
    B_3(\tilde{x}) &= \frac{\tilde{x}^3}{6}
    \end{aligned}
    $$
    [S1, unofficial]
    *Note: The basis functions sum to 1 for any $\tilde{x} \in [0, 1]$ [S1, unofficial].*

4.  **Compute Interpolated Value:** Sum the weighted contributions from the $4 \times 4 \times 4 = 64$ control points in the support region [S1, unofficial]:
    $$
    u(\mathbf{p}) = \sum_{l=0}^{3} \sum_{m=0}^{3} \sum_{n=0}^{3} B_l(\tilde{x}) B_m(\tilde{y}) B_n(\tilde{z}) W_{i_x+l, i_y+m, i_z+n}
    $$
    [S1, unofficial]

**Pitfalls:**
*   **Non-Interpolatory Nature:** The field value at a control point location (where $\tilde{x}=\tilde{y}=\tilde{z}=0$) is not equal to the weight of that control point. The value is a blend of neighboring weights. For example, a single non-zero weight of 1 surrounded by zeros results in a field value of $2/3$ at the control point location [S1, unofficial].
*   **Boundary Conditions:** The indexing formula assumes the query point lies within the bounds defined by the grid origin and spacing. Queries outside the support of the defined grid may require clamping or padding logic not specified in the base method [G].

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
        weights: 3D array of control point weights.
        
    Returns:
        Interpolated scalar value.
    """
    # Step 1: Calculate integer indices
    ix = int((x - x0) / dx - 1)
    iy = int((y - y0) / dy - 1)
    iz = int((z - z0) / dz - 1)
    
    # Step 2: Calculate normalized coordinates
    tx = (x - x0) / dx - 1 - ix
    ty = (y - y0) / dy - 1 - iy
    tz = (z - z0) / dz - 1 - iz
    
    # Step 3: Evaluate basis functions
    bx = cubic_b_spline_basis(tx)
    by = cubic_b_spline_basis(ty)
    bz = cubic_b_spline_basis(tz)
    
    # Step 4: Sum weighted contributions
    value = 0.0
    for l in range(4):
        for m in range(4):
            for n in range(4):
                # Access weight, handling potential boundary checks if necessary [G]
                w = weights[ix + l, iy + m, iz + n]
                value += bx[l] * by[m] * bz[n] * w
                
    return value
```

## Subtopics

### Time-varying B-spline Deformation Field
A variant where the displacement field $u(\mathbf{x}, t)$ is parameterized by cubic B-splines on a fixed 3D spatial grid, but the weights $w(t)$ vary with time [S1, unofficial]. It is chosen when representing dynamic deformations (e.g., in medical imaging or animation) to allow arbitrary continuous position and time queries without discretizing time into separate timesteps [S1, unofficial]. The weights are generated implicitly as a function of time, typically via a Multi-Layer Perceptron (MLP) $\phi(t) \rightarrow w(t)$, which ensures implicit time continuity and efficiency [S1, unofficial]. The spatial interpolation remains identical to the static case, but the weights $W_{i,j,k}$ are replaced by time-dependent weights $W_{i,j,k}(t)$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, as it always accesses exactly 64 control points in 3D regardless of grid size [S1, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the weight grid, where $N$ is the number of control points per dimension [S1, unofficial]. Minimum memory is 64 weights for a scalar field, or 192 weights for a 3D vector field [S1, unofficial].
*   **Strengths:** High continuity ($C^2$), local support ensures stability and efficiency, and the linear combination of weights allows for easy optimization (e.g., via gradient descent) [S1, unofficial].
*   **Limitations:** The field is not interpolatory (values at control points are blended), which may be undesirable if exact data fitting is required [S1, unofficial].
*   **Performance:** Measured performance depends on the hardware's floating-point unit efficiency for cubic polynomial evaluation. In the context of neural rendering for dynamic tomography, it enables efficient ray marching by providing smooth density/deformation fields [S1, unofficial].

## Background
The method is based on cubic B-splines introduced by Rueckert et al. [19] (cited in [S1, unofficial]). It was adapted for neural rendering and dynamic tomography to represent continuous deformation and density fields, allowing for the reconstruction of 3D structures from X-ray projections without voxelization [S1, unofficial]. The specific indexing and basis function formulas are detailed in the appendix of the source paper [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
