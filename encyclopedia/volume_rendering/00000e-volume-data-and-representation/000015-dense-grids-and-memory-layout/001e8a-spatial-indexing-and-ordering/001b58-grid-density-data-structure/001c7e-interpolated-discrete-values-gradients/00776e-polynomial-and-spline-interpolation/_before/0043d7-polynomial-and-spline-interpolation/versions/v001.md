# Polynomial and Spline Interpolation

Also called: Hermite Interpolation, B-spline Interpolation

## Overview
Polynomial and Spline Interpolation provides continuous, differentiable scalar fields from discrete grid data by evaluating weighted sums of basis functions, enabling smooth shading and accurate gradient estimation in volume rendering. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md) which yields discontinuous $C^0$ approximations, or [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) which uses local quadratic segments, this family employs higher-order polynomials (cubic) over wider support regions to guarantee $C^1$ or $C^2$ continuity and richer visual appearance [S1, unofficial].

## Shared principles
All methods in this family map a query point $P$ in continuous space to a discrete set of control points (grid vertices) and compute a weighted sum of their values using basis functions $B_k$. The core mechanism involves:
1.  **Indexing:** Determining the integer indices of the support grid surrounding the query point.
2.  **Normalization:** Computing local coordinates $\tilde{x}, \tilde{y}, \tilde{z}$ within the unit cell or support region.
3.  **Basis Evaluation:** Calculating weights using polynomial basis functions that satisfy specific continuity and partition-of-unity properties.
4.  **Blending:** Summing the products of basis weights and control point values.

The choice of basis function determines the continuity order ($C^0, C^1, C^2$) and the support width (number of neighboring points influencing the result).

## Subtopics

### [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md)
This method interpolates a normalized scalar field within a 1D interval using two piecewise quadratic Bézier curves to satisfy Hermite data (value and derivative) at both endpoints. It solves the incompatibility of single quadratic interpolation with four constraints (two values, two derivatives) by splitting the interval, providing exact interpolation in specific cases and better stability than cubic interpolation for small subdivision counts. See [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) for implementation details.

### [B-spline Field Interpolation](11066-b-spline-field-interpolation.md)
B-spline Field Interpolation is a method for evaluating a continuous function from a discrete grid of control points using cubic B-splines, where shape functions have local influence. It solves the problem of querying a smooth, differentiable field at arbitrary continuous coordinates by blending the weights of a fixed 4×4×4 support grid. It is distinguished from its siblings by its use of cubic polynomial basis functions that sum to 1, providing $C^2$ continuity, and by the specific indexing scheme that maps query points to a local 64-point support region. Unlike [Nearest Voxel Lookup with Periodic Boundary](1573-nearest-voxel-lookup-with-periodic-boundary.md) which uses discrete selection, or [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) which uses quadratic curves, this method uses cubic polynomials over a wider support. See [B-spline Field Interpolation](11066-b-spline-field-interpolation.md) for implementation details.

### Hermite Interpolation of Scalar Fields
A technique for rendering scalar valued functions (height and signed distance fields) by storing partial derivatives with function values and using Hermite interpolation between samples to guarantee a globally $C^1$-continuous result [S1, unofficial]. It provides a storage efficient replacement for normal mapping with richer visual appearance, as it does not necessarily require extra storage if surface normals are already part of the discrete field of samples [S1, unofficial]. The method involves storing partial derivatives of the scalar valued function with the function values, using Hermite interpolation between the samples, and using the exact normals of the reconstructed cubic Hermite surface as shading normals [S1, unofficial]. It generalizes to arbitrary orders and dimensions, and derives an approximation for mixed partial derivatives for three dimensional first order fields, akin to Adini's method for parametric surface patches [S1, unofficial].

## Background
Hermite interpolation in computer graphics was proposed to replace standard trilinear filtering with $C^1$-continuous results, allowing the use of exact surface normals for shading [S1, unofficial]. The approach was demonstrated in the context of accelerating sphere tracing and rendering height fields and signed distance fields [S1, unofficial]. B-spline interpolation is a standard technique in computer graphics for smooth surface modeling, adapted here for dense volume data representation [S1, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
