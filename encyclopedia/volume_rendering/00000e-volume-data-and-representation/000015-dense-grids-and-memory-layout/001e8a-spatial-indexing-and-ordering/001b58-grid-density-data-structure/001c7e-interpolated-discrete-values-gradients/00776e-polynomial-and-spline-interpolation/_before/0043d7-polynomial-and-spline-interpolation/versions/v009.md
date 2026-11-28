# Polynomial and Spline Interpolation

Also called: Continuous Field Reconstruction from Discrete Grids

## Overview
Polynomial and Spline Interpolation provides a continuous, differentiable scalar or vector field $u(\mathbf{p})$ from discrete grid data, enabling smooth volume rendering and gradient-based shading. It is distinguished from nearest-neighbor or simple linear methods by its higher-order continuity ($C^1$ or $C^2$) and from global basis methods by its local support, ensuring that the value at any query point depends only on a finite neighborhood of grid indices.

## Shared principles
The family shares the mechanism of evaluating a continuous function $u(\mathbf{p})$ at arbitrary query coordinates $\mathbf{p}$ using a weighted sum of basis functions centered on a local support region of the grid. The general form is:
$$ u(\mathbf{p}) = \sum_{k} w_k B_k(\mathbf{p}) $$
where $w_k$ are the grid values (or control points) and $B_k$ are the basis functions. Key shared constraints include:
1.  **Local Support**: Basis functions have compact support, meaning $B_k(\mathbf{p}) = 0$ outside a small neighborhood of grid indices. This limits computation to a fixed number of neighbors (e.g., 8 for trilinear, 64 for cubic B-spline).
2.  **Partition of Unity**: For interpolation stability, the basis functions typically sum to one: $\sum_k B_k(\mathbf{p}) = 1$.
3.  **Coordinate Mapping**: Query points must be mapped from world space to grid indices using the grid origin $\mathbf{o}$ and spacing $\mathbf{s}$: $\mathbf{idx} = \lfloor (\mathbf{p} - \mathbf{o}) / \mathbf{s} \rfloor$.
4.  **Boundary Handling**: Out-of-bounds accesses require explicit strategies (periodic, clamp, or constant) as the basis functions do not inherently define values outside the grid domain.

## Subtopics

### [Grid Cell Collection via Corner Indices](1518-grid-cell-collection-via-corner-indices.md)
Extracts the 8 vertices of the enclosing grid cell. It is the foundational data retrieval step for linear and higher-order interpolation, solving the mapping from continuous coordinates to discrete tensor indices. Choose this when implementing trilinear interpolation or surface extraction methods that require explicit access to corner values.

### [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md)
Uses two piecewise quadratic Bézier curves to satisfy Hermite data (value and derivative) at endpoints. It solves the incompatibility of single quadratic interpolation with four constraints, providing $C^1$ continuity with lower computational cost than cubic interpolation. Choose this for root isolation or 1D field reconstruction where derivative information is available.

### [B-spline Field Interpolation](11066-b-spline-field-interpolation.md)
Evaluates a continuous function using cubic B-spline basis functions over a $4 \times 4 \times 4$ support. It provides $C^2$ continuity and local support, distinguishing it from linear methods by smoothness and from global methods by locality. Choose this for deformation networks or smooth volume rendering where $C^2$ continuity is required.

### [Voxel Data Interpolation and Sampling](12781-voxel-data-interpolation-and-sampling.md)
Retrieves scalar values and gradients to drive transfer functions. It differs by explicitly coupling scalar sampling with optional gradient magnitude extraction. Choose this when the rendering pipeline requires transfer function inputs based on both value and edge strength.

### Hermite Interpolation of Scalar Fields
- **Definition**: A technique for rendering scalar valued functions (height and signed distance fields) by storing partial derivatives with function values and using Hermite interpolation between samples to guarantee a globally $C^1$-continuous result [S1, unofficial].
- **Steps**: Store the partial derivatives of the scalar valued function with the function values. Use Hermite interpolation between the samples. Use the exact normals of the reconstructed cubic Hermite surface as shading normals. Derive an approximation for mixed partial derivatives for three dimensional first order fields, akin to Adini's method for parametric surface patches [S1, unofficial].
- **Comparison**: Provides a storage efficient replacement for normal mapping with richer visual appearance, as it does not necessarily require extra storage if surface normals are already part of the discrete field of samples [S1, unofficial].

### Box Spline Interpolation
- **Definition**: A piecewise polynomial interpolation function with finite support, uniquely defined by a direction matrix, used for reconstructing continuous splines from discrete datasets [S2].
- **Steps**: Construct the box-spline by taking consecutive directional convolutions along each column direction of the direction matrix $\Xi$. For non-Cartesian lattices, apply a change-of-variables to the Cartesian lattice theory [S2].
- **Formula**: Recursive definition: $M_{\Xi \cup \{\mathbf{d}\}}(\mathbf{x}) = \int_0^1 M_\Xi(\mathbf{x} - t\mathbf{d}) dt, \mathbf{x} \in \mathbb{R}^n$ (Eq 6) [S2].
- **Comparison**: Provides higher quality and reasonable performance compared to trilinear or tricubic interpolations commonly used on mobile devices [S2].
- **Usage**: Mobile volume rendering, FCC datasets, GPU ray casting [S2].

### Distance Weighting Filter
- **Definition**: A spatial interpolation filter that assigns weights based on distance to preserve data structure during grid mapping [S3, unofficial].
- **Parameter**: Filter radius determines the spatial extent of the weighting; must be shortened to increase sampling rate [S3, unofficial].
- **Pitfall**: Assumes a constant, second-order distribution of weighting terms based on the filter radius; under-samples areas with high data frequency, causing inner and outer shells to disappear where data density varies [S3, unofficial].
- **Comparison**: Performs worse than the Gaussian filter for preserving accuracy because it lacks a degree of freedom for adjusting sampling frequency independently of radius [S3, unofficial].

## Background
Hermite interpolation for scalar fields was proposed to provide $C^1$-continuous results for real-time computer graphics, generalizing to arbitrary orders and dimensions [S1, unofficial]. Box splines, introduced by de Boor et al., are multivariate generalizations of uniform B-splines used for real-time volume visualization on mobile platforms with fixed capabilities, specifically for FCC datasets [S2]. Distance weighting filters were analyzed in the context of sampling scattered data onto rectangular grids for volume visualization, noting limitations in preserving accuracy for variable density data [S3, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S2] Real-Time Volume Rendering of FCC Datasets Using Box-Splines on the Mobile Platforms, https://hal.science/hal-05467237v1, fetched 2026-10-27
[S3] Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
