# Polynomial and Spline Interpolation

Also called: Continuous Field Reconstruction from Discrete Grids

## Overview
Polynomial and Spline Interpolation provides scalar value and gradient fields computed from discrete data at grid vertices using continuous basis functions, enabling smooth evaluation at arbitrary coordinates [S1, 11066-b-spline-field-interpolation.md, unofficial]. It is distinguished from its parent [Interpolated Discrete Values & Gradients](1268-interpolated-discrete-values-gradients.md) by its explicit use of higher-order continuity (e.g., $C^1$ or $C^2$) and local support properties, and from its sibling [Data Interpolation](7180-data-interpolation.md) by its focus on piecewise polynomial bases (Hermite, B-splines) rather than general kernel methods.

## Shared principles
The family shares the mechanism of evaluating a continuous function $u(\mathbf{p})$ at a query point $\mathbf{p}$ by summing weighted contributions from a finite set of discrete control points (grid vertices) within a local support region. The value is computed as:
$$ u(\mathbf{p}) = \sum_{k} w_k B_k(\mathbf{p}) $$
where $w_k$ are the discrete values (or derivatives) stored at grid locations, and $B_k(\mathbf{p})$ are basis functions determined by the specific interpolation method. Key shared properties include:
1.  **Local Support:** Only a fixed number of grid points influence the result at any query location, ensuring $O(1)$ evaluation complexity relative to grid size [11066-b-spline-field-interpolation.md].
2.  **Continuity:** The resulting field is continuous across cell boundaries, with the order of continuity ($C^0, C^1, C^2$, etc.) determined by the basis function [S1, 11066-b-spline-field-interpolation.md, unofficial].
3.  **Coordinate Transformation:** Query points are mapped to local normalized coordinates within the support region to evaluate the basis functions [11066-b-spline-field-interpolation.md].

## Subtopics

### [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md)
A method for interpolating a normalized scalar field within a 1D interval using two piecewise quadratic Bézier curves to satisfy Hermite data (value and derivative) at both endpoints [4001-piecewise-quadratic-hermite-interpolation.md]. It solves the incompatibility of single quadratic interpolation with four constraints (two values, two derivatives) by splitting the interval, providing a $C^1$ continuous interpolant that is more numerically stable and efficient than cubic interpolation, particularly for root isolation in volume rendering [4001-piecewise-quadratic-hermite-interpolation.md]. Choose this when $C^1$ continuity is required but cubic overhead is prohibitive, or when exact derivative matching at endpoints is critical.

### [B-spline Field Interpolation](11066-b-spline-field-interpolation.md)
B-spline Field Interpolation is a method for evaluating continuous scalar or vector fields from a discrete grid of control points using cubic B-spline basis functions [11066-b-spline-field-interpolation.md]. It solves the problem of querying a smooth, differentiable field at arbitrary continuous coordinates while maintaining local support, meaning only a fixed, small number of grid points influence any single query [11066-b-spline-field-interpolation.md]. It is distinguished from its parent [Polynomial and Spline Interpolation](11104-polynomial-and-spline-interpolation.md) by its specific use of cubic B-splines, which provide $C^2$ continuity and a fixed 4x4x4 support kernel in 3D, and from its sibling [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) by its higher-order continuity and larger support footprint [11066-b-spline-field-interpolation.md]. Choose this when $C^2$ smoothness is required or when vector field interpolation is needed.

### Hermite Interpolation of Scalar Fields
**Definition:** A technique for rendering scalar valued functions (height and signed distance fields) by storing partial derivatives with function values and using Hermite interpolation between samples to guarantee a globally $C^1$-continuous result [S1, unofficial].
**Steps:** Store the partial derivatives of the scalar valued function with the function values. Use Hermite interpolation between the samples. Use the exact normals of the reconstructed cubic Hermite surface as shading normals. Derive an approximation for mixed partial derivatives for three dimensional first order fields, akin to Adini's method for parametric surface patches [S1, unofficial].
**Comparison:** Provides a storage efficient replacement for normal mapping with richer visual appearance, as it does not necessarily require extra storage if surface normals are already part of the discrete field of samples [S1, unofficial].

### Box Spline Interpolation
**Definition:** A piecewise polynomial interpolation function with finite support, uniquely defined by a direction matrix, used for reconstructing continuous splines from discrete datasets [S2].
**Steps:** Construct the box-spline by taking consecutive directional convolutions along each column direction of the direction matrix $\Xi$. For non-Cartesian lattices, apply a change-of-variables to the Cartesian lattice theory [S2].
**Formula:** Recursive definition: $M_{\Xi \cup \{\mathbf{d}\}}(\mathbf{x}) = \int_0^1 M_\Xi(\mathbf{x} - t\mathbf{d}) dt, \mathbf{x} \in \mathbb{R}^n$ (Eq 6) [S2].
**Comparison:** Provides higher quality and reasonable performance compared to trilinear or tricubic interpolations commonly used on mobile devices [S2].
**Usage:** Mobile volume rendering, FCC datasets, GPU ray casting [S2].

### Distance Weighting Filter
**Definition:** A spatial interpolation filter that assigns weights based on distance to preserve data structure during grid mapping [S3, unofficial].
**Parameter:** Filter radius determines the spatial extent of the weighting; must be shortened to increase sampling rate [S3, unofficial].
**Pitfall:** Assumes a constant, second-order distribution of weighting terms based on the filter radius; under-samples areas with high data frequency, causing inner and outer shells to disappear where data density varies [S3, unofficial].
**Comparison:** Performs worse than the Gaussian filter for preserving accuracy because it lacks a degree of freedom for adjusting sampling frequency independently of radius [S3, unofficial].

## Background
Hermite interpolation in volume rendering was proposed to address the $C^0$ discontinuities of standard trilinear filtering, offering a globally $C^1$-continuous result by storing partial derivatives alongside function values [S1, unofficial]. This approach allows the use of exact normals from the reconstructed cubic Hermite surface as shading normals, providing a storage-efficient replacement for normal mapping [S1, unofficial]. The method generalizes to arbitrary orders and dimensions and includes approximations for mixed partial derivatives in 3D fields [S1, unofficial].

Box splines, introduced by de Boor et al. as multivariate generalizations of uniform B-splines, have been adapted for real-time volume visualization on mobile devices. Specifically, 6-direction cubic box-splines on Face-Centered Cubic (FCC) datasets were implemented to achieve high quality and reasonable performance where traditional trilinear or tricubic interpolations were insufficient [S2].

Distance weighting filters have been used in A-buffer sampling to preserve data structure during grid mapping, though they are noted to under-sample high-frequency areas compared to Gaussian filters [S3, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S2] Real-Time Volume Rendering of FCC Datasets Using Box-Splines on the Mobile Platforms, https://hal.science/hal-05467237v1, fetched 2026-10-27
[S3] Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
