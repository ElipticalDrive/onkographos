# B-spline Field Interpolation

Also called: B-spline deformation field, B-spline interpolation, Cubic B-spline field, Cubic B-spline interpolation for heterogeneous refractive fields, Heterogeneous refractive field representation

## Overview
B-spline Field Interpolation is a method for function interpolation using cubic B-splines, defined by a grid of control points and weights, where shape functions have local influence [S2, unofficial]. It solves the problem of evaluating a continuous scalar or vector field from discrete control points with $C^2$ continuity, ensuring that constant fields are reproduced exactly via partition of unity [G]. It sets itself apart from linear voxel interpolation by using a $4 \times 4 \times 4$ support region and cubic basis functions, providing smoother gradients and higher-order continuity [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Query point $\mathbf{p} = (x, y, z)$; grid origin $\mathbf{x}_0 = (x_0, y_0, z_0)$; grid spacing $\mathbf{dx} = (dx, dy, dz)$; weight tensor $\mathbf{W}$ of shape $(N_x, N_y, N_z, C)$, where $C$ is the channel count (e.g., 1 for scalar, 3 for displacement) [S2, unofficial].
*   **Outputs:** Interpolated value $u(\mathbf{p})$ (scalar or vector) [S2, unofficial].

**Data Structures:**
The weights are stored in a dense 3D grid [S2, unofficial]. For a 3D field, the minimum number of control points is $4 \times 4 \times 4 = 64$ [S2, unofficial]. For vector fields (e.g., displacement), 3 sets of weights are required, totaling 192 degrees of freedom [S2, unofficial].

**Steps:**
1.  **Index Calculation:** Determine the integer indices of the $4 \times 4 \times 4$ support grid surrounding the query point [S2, unofficial]. The indices $i_x, i_y, i_z$ are calculated as:
    $$ i_x = \left\lfloor \frac{x - x_0}{dx} - 1 \right\rfloor, \quad i_y = \left\lfloor \frac{y - y_0}{dy} - 1 \right\rfloor, \quad i_z = \left\lfloor \frac{z - z_0}{dz} - 1 \right\rfloor $$
    [S2, unofficial]
2.  **Normalized Coordinates:** Calculate the fractional coordinates $\tilde{x}, \tilde{y}, \tilde{z}$ within the local support cell:
    $$ \tilde{x} = \frac{x - x_0}{dx} - 1 - i_x, \quad \tilde{y} = \frac{y - y_0}{dy} - 1 - i_y, \quad \tilde{z} = \frac{z - z_0}{dz} - 1 - i_z $$
    [S2, unofficial]
3.  **Basis Evaluation:** Evaluate the cubic B-spline basis functions $B_0, B_1, B_2, B_3$ for each dimension using $\tilde{x}, \tilde{y}, \tilde{z}$ [S2, unofficial]:
    $$ B_0(\tilde{x}) = \frac{(1 - \tilde{x})^3}{6} $$
    $$ B_1(\tilde{x}) = \frac{3\tilde{x}^3 - 6\tilde{x}^2 + 4}{6} $$
    $$ B_2(\tilde{x}) = \frac{-3\tilde{x}^3 + 3\tilde{x}^2 + 3\tilde{x} + 1}{6} $$
    $$ B_3(\tilde{x}) = \frac{\tilde{x}^3}{6} $$
    [S2, unofficial]
4.  **Weighted Summation:** Compute the interpolated value by summing the products of basis functions and weights over the $4 \times 4 \times 4$ support region:
    $$ u(\mathbf{p}) = \sum_{l=0}^3 \sum_{m=0}^3 \sum_{n=0}^3 B_l(\tilde{x}) B_m(\tilde{y}) B_n(\tilde{z}) w_{i_x+l, i_y+m, i_z+n} $$
    [S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def b_spline_interpolate(x, y, z, x0, y0, z0, dx, dy, dz, weights):
    """
    Interpolate value at (x, y, z) using cubic B-splines.
    weights: numpy array of shape (Nx, Ny, Nz, C)
    """
    # 1. Calculate integer indices
    ix = math.floor((x - x0) / dx - 1)
    iy = math.floor((y - y0) / dy - 1)
    iz = math.floor((z - z0) / dz - 1)

    # 2. Calculate normalized coordinates
    x_tilde = (x - x0) / dx - 1 - ix
    y_tilde = (y - y0) / dy - 1 - iy
    z_tilde = (z - z0) / dz - 1 - iz

    # 3. Evaluate basis functions
    def b0(t): return (1 - t)**3 / 6
    def b1(t): return (3*t**3 - 6*t**2 + 4) / 6
    def b2(t): return (-3*t**3 + 3*t**2 + 3*t + 1) / 6
    def b3(t): return t**3 / 6

    bx = [b0(x_tilde), b1(x_tilde), b2(x_tilde), b3(x_tilde)]
    by = [b0(y_tilde), b1(y_tilde), b2(y_tilde), b3(y_tilde)]
    bz = [b0(z_tilde), b1(z_tilde), b2(z_tilde), b3(z_tilde)]

    # 4. Weighted summation
    result = 0.0
    for l in range(4):
        for m in range(4):
            for n in range(4):
                w = weights[ix + l, iy + m, iz + n]
                result += bx[l] * by[m] * bz[n] * w
    return result
```

**Parameters:**
*   Grid origin: $[x_0, y_0, z_0]$ [S2, unofficial]
*   Grid spacing: $dx, dy, dz$ [S2, unofficial]
*   Control point weight: $w_{i_x, i_y, i_z}$ [S2, unofficial]

**Edge Cases and Pitfalls:**
*   The field value at a control point ($\tilde{x}=0$) is not equal to the weight of that control point due to the blending of neighboring weights [S2, unofficial]. For example, with weights $[0, 0, 0, 1, 0, 0, 0]$ and origin $x_0=-2, dx=0.5$, the value at $x=0$ is $2/3$, not $1$ [S2, unofficial].
*   Boundary conditions are not explicitly defined in the core method; queries outside the grid bounds require handling (e.g., clamping or extrapolation) [G].

## Subtopics

### [B-spline Curve Modeling](14651-b-spline-curve-modeling.md)
B-spline Curve Modeling is a technique for generating smooth parametric space curves that interpolate a sequence of control points, primarily used to define the spines of volumetric flames. It is chosen over Hermite interpolation because it guarantees second-order continuity ($C^2$), preventing random twisting of the curve space, and possesses affine invariance. It builds on [B-spline Field Interpolation](11066-b-spline-field-interpolation.md) by applying uniform B-spline basis functions to 1D parametric curves rather than 3D scalar fields [G].

### Time-varying B-spline Deformation Field
A time-varying displacement field $u(x, t)$ parametrized by cubic B-splines on a fixed 3D grid, allowing arbitrary continuous position and time queries [S2, unofficial]. It is chosen when dynamic deformation needs to be represented efficiently without discretizing time into timesteps, building in implicit time continuity [S2, unofficial]. It builds on [B-spline Field Interpolation](11066-b-spline-field-interpolation.md) by making the weights $w(t)$ time-dependent, generated by an MLP $\phi(t) \to w(t)$ [S2, unofficial]. The displacement component $u$ at $(x, y, z)$ and time $t$ is calculated by interpolating weights of the 4 closest grid points in each dimension: $u(x, y, z, t) = \sum_{l=0}^3 \sum_{m=0}^3 \sum_{n=0}^3 B_l(\tilde{x}) B_m(\tilde{y}) B_n(\tilde{z}) w(t)_{i_x+l, i_y+m, i_z+n}$ [S2, unofficial]. The grid has size $n_x \times n_y \times n_z$, with trainable weights for each of the 3 displacement components [S2, unofficial].

### [Cox-de Boor recursion for B-spline basis functions](16031-cox-de-boor-recursion-for-b-spline-basis-functions.md)
The Cox-de Boor recursion is a numerically stable algorithm for evaluating B-spline basis functions, serving as the computational core for B-spline curve and surface evaluation. It solves the problem of efficiently computing the shape functions that define the local influence of control points in a B-spline field. It sets itself apart from direct polynomial evaluation by using a recursive linear combination of lower-degree basis functions, which avoids numerical instability and leverages the local support property of B-splines.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, as it always accesses a fixed $4 \times 4 \times 4 = 64$ support region [S2, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z C)$, where $N$ are grid dimensions and $C$ is channel count [S2, unofficial]. Minimum 64 points for scalar, 192 for vector [S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Local influence of shape functions, $C^2$ continuity, partition of unity, efficient for dynamic fields via time-varying weights [S2, unofficial].
*   **Limitations:** Field value at control points does not equal weight; requires careful handling of boundaries [S2, unofficial].
*   **When to use:** When smooth, continuous interpolation with local control is needed, especially for deformation fields or dynamic scenes [S2, unofficial].
*   **When not:** When exact interpolation of control points is required (use Lagrange or other methods) [S2, unofficial].

## Background
Originated in the context of spline-based deformation networks for medical imaging and tomography, specifically modified to account for temporal variations in deformation [S2, unofficial]. The method is based on cubic B-splines by Rueckert et al. [19] (cited in [S2, unofficial]). It is used in dynamic tomography and X-ray attenuation rendering to represent continuous density or displacement fields [S2, unofficial]. It is also applied to represent heterogeneous refractive fields for path tracing estimators [S1].

## Sources
[S1] Path tracing estimators for refractive radiative transfer, https://doi.org/10.1145/3414685.3417793, fetched 2026-10-29
[S2] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
