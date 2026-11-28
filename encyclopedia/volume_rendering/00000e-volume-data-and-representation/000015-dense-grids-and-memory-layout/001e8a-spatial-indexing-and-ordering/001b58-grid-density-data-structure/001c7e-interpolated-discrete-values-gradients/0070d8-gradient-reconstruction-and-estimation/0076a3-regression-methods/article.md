# Regression Methods

Also called: 3D/4D regression, Weighted/Unweighted regression

## Overview
Regression methods approximate the continuous gradient $\nabla f(\mathbf{v})$ by fitting a local polynomial model (typically a plane) to the scalar values of neighboring voxels. Unlike averaging methods which are generally more accurate on high-quality meshes, regression methods are chosen for their adaptability to low-quality, irregular, or adaptive meshes where uniform weighting fails [S1].

## Shared principles
All methods in this family share the fundamental goal of approximating the continuous gradient $\nabla f(\mathbf{v})$ from discrete samples. The core mechanism involves combining scalar field values from neighboring voxels using a basis function or difference kernel to produce a vector output.

1.  **Linear Combination of Neighbors**: The gradient at a query point $\mathbf{v}$ is typically computed as a linear combination of scalar values (or random variables representing values) from neighboring voxels $\mathbf{v}_i$.
    $$ \nabla f(\mathbf{v}) \approx \sum_i w_i \cdot f(\mathbf{v}_i) $$
    where $w_i$ are weights derived from the derivative of the interpolation basis function.

2.  **Basis Function Derivation**: The weights are often derived from a derivative reconstruction filter $\rho$, obtained by convolving a continuous interpolation filter with a digital derivative filter [S1]. For example, in standard Cartesian grids, this results in central difference schemes.

3.  **Support Limitation**: Only neighbors within the support of the basis function $\rho$ contribute to the gradient. Voxels outside this support are assigned zero weights [S1].

4.  **Uncertainty Propagation**: When the input scalar field is uncertain (modeled as random variables $X_i$), the output gradient is a trivariate random vector $\mathbf{Y} = (Y_x, Y_y, Y_z)^T$. The distribution of $\mathbf{Y}$ can be derived analytically using spline frameworks; for instance, if input uncertainties are uniform, the gradient components follow box spline distributions [S1].

## Subtopics

### [Fixed-Size Neighborhood Reconstruction](23574-fixed-size-neighborhood-reconstruction.md)
A gradient reconstruction method that uses a fixed number of neighbors ($K$) instead of all connectivity neighbors, allowing for efficient GPU encoding and on-the-fly computation for dynamic meshes.

## Background
Regression-based gradient estimation originated in the context of computational fluid dynamics (CFD) and finite-element analysis, where unstructured meshes are common. It was adapted for volume rendering on unstructured meshes to handle cases where traditional averaging methods degrade due to mesh irregularity [S1].

The method is categorized by the dimensionality of the regression model:
*   **3D Regression**: Fits a plane to cell values, estimating the gradient components directly [S1].
*   **4D Regression**: Includes an additional dimension, typically a constant term or bias, in the model [S1].

Weighted regression variants, such as inverse centroid distance weighting, are preferred over unweighted regression because unweighted versions are highly sensitive to mesh irregularity. On highly irregular or low-quality meshes, unweighted regression produces poor gradient reconstructions that manifest as visible creases, folds, and artificial flatness in volume rendering [S1]. Weighting adds negligible computational cost compared to unweighted regression while significantly improving accuracy [S1].

While averaging methods (such as Green-Gauss or inverse centroid distance) generally provide the most accurate gradient reconstruction on high-quality, regular meshes, regression methods adapt better to low-quality or adaptive meshes [S1].

## Sources
[S1] A Comparison of Gradient Estimation Methods for Volume Rendering on Unstructured Meshes, https://www.computer.org/csdl/journal/tg/2011/03/ttg2011030305/13rRUx0xPi5, fetched 2026-11-01
