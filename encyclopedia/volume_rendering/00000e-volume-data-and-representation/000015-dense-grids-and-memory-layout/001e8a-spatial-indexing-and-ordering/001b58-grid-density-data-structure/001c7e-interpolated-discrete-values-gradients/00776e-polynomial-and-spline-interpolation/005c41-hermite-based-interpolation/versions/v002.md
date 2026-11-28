# Hermite-based Interpolation

Also called: Hermite interpolation, Gradient-based interpolation

## Overview
Hermite-based interpolation constructs a continuous, differentiable scalar field by interpolating both function values and their spatial derivatives (gradients) stored at grid vertices . It differs from its siblings by explicitly utilizing gradient information to guarantee $C^1$ continuity, allowing the exact normals of the reconstructed surface to be used as shading normals, which provides a storage-efficient replacement for normal mapping .

## Shared principles
The family operates on a dense grid tensor $\mathbf{G}$ of shape $(D_x, D_y, D_z, C)$, where $D$ are spatial dimensions and $C$ is the channel count (e.g., density, color, or geometry parameters) . The core mechanism involves mapping a continuous query point $\mathbf{p} = (x, y, z)$ to a local support region of discrete control points or vertices, evaluating basis functions over this region, and computing a weighted sum .

1.  **Coordinate Mapping:** Query coordinates are normalized or mapped to the grid's indexing space. For a grid with origin $\mathbf{x}_0$ and spacing $\mathbf{dx}$, the continuous coordinate is transformed to a local fractional coordinate $\tilde{\mathbf{x}}$ within a unit cell or support window .
2.  **Support Selection:** A subset of grid vertices is selected based on the query position.
    *   For linear/trilinear methods, this is the 8 corners of the enclosing voxel .
    *   For cubic B-splines, this is the $4 \times 4 \times 4$ block of control points surrounding the query point .
3.  **Basis Evaluation:** Weighting coefficients are computed using basis functions $B_i(\tilde{x})$. These functions must satisfy specific properties:
    *   **Partition of Unity:** $\sum B_i(\tilde{x}) = 1$ to ensure constant fields are reproduced exactly .
    *   **Continuity:** The degree of the polynomial determines the continuity order ($C^0$ for linear, $C^2$ for cubic B-splines) .
4.  **Weighted Summation:** The interpolated value $u(\mathbf{p})$ is calculated as:
    $$ u(\mathbf{p}) = \sum_{i} \sum_{j} \sum_{k} B_i(\tilde{x}) B_j(\tilde{y}) B_k(\tilde{z}) w_{i,j,k} $$
    where $w_{i,j,k}$ are the values or weights at the selected support points .
5.  **Gradient Computation:** Gradients are derived by differentiating the basis functions with respect to the query coordinates, allowing for the computation of $\nabla u(\mathbf{p})$ .

## Subtopics

### [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md)
A method for interpolating a normalized scalar field within a 1D interval using two connected quadratic Bézier curves, ensuring $C^1$ continuity while respecting Hermite data (value and derivative) at both endpoints. It solves the incompatibility of single quadratic interpolation with four constraints (two values, two derivatives) by splitting the interval, providing a numerically stable and efficient alternative to cubic interpolation for root isolation in volume rendering. It is specifically designed for use within the SCALIS framework, where it operates on a normalized field $g \circ f \circ r^{-1}$ to improve convergence for small numbers of subdivisions compared to cubic interpolation on the original field.

### [Hermite Interpolation of Scalar Fields](9450-hermite-interpolation-of-scalar-fields.md)
Renders scalar-valued functions (such as height fields or signed distance fields) by storing partial derivatives alongside function values and using Hermite interpolation between samples to guarantee a globally $C^1$-continuous result. It differs from its siblings by explicitly utilizing gradient information to construct a cubic Hermite surface, allowing the exact normals of this surface to be used as shading normals, which provides a storage-efficient replacement for normal mapping with richer visual appearance.

### [Third Order Elimination (Scaling)](13712-third-order-elimination-scaling.md)
A quadratic interpolation method derived from cubic Hermite interpolation by scaling the input derivatives to force the cubic coefficient ($a_3$) to zero. It solves the problem of approximating a smooth field with a polynomial that retains quadratic continuity while strictly interpolating the function values at grid vertices. It differs from its sibling [Hermite Interpolation of Scalar Fields](9450-hermite-interpolation-of-scalar-fields.md) by discarding the cubic term entirely through derivative scaling, and from [Tricubic Interpolation](14963-tricubic-interpolation.md) by reducing the polynomial degree to ensure $C^1$ continuity without the complexity of cubic basis evaluation.

### [Tricubic Interpolation](14963-tricubic-interpolation.md)
A higher-order method that employs cubic polynomials over a $4 \times 4 \times 4$ neighborhood of 64 points to approximate smooth underlying functions. It provides $C^1$ continuity and fourth-order accuracy $O(h^4)$, making it superior to trilinear interpolation for gradient-sensitive tasks like volume rendering, albeit at increased computational cost.

## Background
Hermite interpolation is a classical numerical analysis technique for constructing piecewise polynomial functions that interpolate both function values and derivatives. In the context of volume rendering and computer graphics, it has been adapted to handle 3D scalar fields, often referred to as Hermite splines or gradient-based interpolation. The specific application to rendering scalar fields for $C^1$-continuous shading and normal estimation is detailed in [Hermite Interpolation of Scalar Fields](9450-hermite-interpolation-of-scalar-fields.md). The piecewise quadratic variant is utilized in the SCALIS framework for robust root isolation . The third-order elimination method offers a computationally cheaper quadratic alternative .

## Sources
