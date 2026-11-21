# Covariance Stabilization and Correction
## Overview
This family addresses the numerical instability and geometric inaccuracies inherent in projecting 3D Gaussian primitives into screen space for differentiable rendering. It solves the problem of aliasing artifacts caused by undersampled Gaussians and the discrepancy between analytically projected covariances and the true integrated 2D covariance. It sets itself apart from its siblings by explicitly modifying the covariance matrix—either through low-pass filtering with density preservation or via learned linear corrections—rather than relying solely on geometric projection or bounding box heuristics.
## Shared principles
All methods in this family operate on the 2D screen-space covariance matrix $\Sigma^{2D}$ derived from a 3D Gaussian. The core mechanism involves modifying the entries of this matrix to ensure numerical stability (positive definiteness) or to correct for projection errors. The output is typically a corrected covariance matrix or its inverse (conic matrix) used in the rasterization kernel. The methods assume the input covariance is symmetric and positive semi-definite.
## Subtopics

### [2D Gaussian Covariance Stabilization and Rescaling](3467-2d-gaussian-covariance-stabilization-and-rescaling.md)
This method stabilizes the projection by applying a low-pass filter to the 2D covariance matrix, ensuring each Gaussian spans at least the width of a single pixel to reduce aliasing [3467-2d-gaussian-covariance-stabilization-and-rescaling]. It computes a rescale factor to preserve the integrated density of the Gaussian after filtering. Choose this when aliasing is a primary concern and density preservation is required. See [2D Gaussian Covariance Stabilization and Rescaling](3467-2d-gaussian-covariance-stabilization-and-rescaling.md).

### [2D Covariance Correction via Linear Regression](9912-2d-covariance-correction-via-linear-regression.md)
This method approximates the relationship between projected 2D covariances and actual 2D covariances using a linear regression model, serving as a computationally efficient alternative to neural network-based corrections [9912-2d-covariance-correction-via-linear-regression]. It sets itself apart by replacing complex learned mappings with closed-form linear coefficients or a single scalar factor, reducing training overhead. Choose this when reconstruction quality comparable to Half-Cosine kernels is needed with lower computational cost. See [2D Covariance Correction via Linear Regression](9912-2d-covariance-correction-via-linear-regression.md).

### Covariance Correction Factor
#### definition
A scalar correction factor applied to the covariance matrix to approximate the relationship between projected 2D covariances and actual 2D covariances, replacing a more complex MLP or Simpson's rule integration [S1].
#### parameter
Inverse Multiquadric correction factor: 1.38f [S1].
Parabolic correction factor: 1.3f [S1].

## Background
Originating from work on generalizing splatting with decaying anisotropic radial basis functions, these techniques were introduced to improve the fidelity of Gaussian splatting rendering [S1]. The stabilization approach addresses the common issue of "sparkles" or aliasing in novel view synthesis, while the correction methods address the geometric mismatch between the projected Gaussian and the true integrated density.
## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
