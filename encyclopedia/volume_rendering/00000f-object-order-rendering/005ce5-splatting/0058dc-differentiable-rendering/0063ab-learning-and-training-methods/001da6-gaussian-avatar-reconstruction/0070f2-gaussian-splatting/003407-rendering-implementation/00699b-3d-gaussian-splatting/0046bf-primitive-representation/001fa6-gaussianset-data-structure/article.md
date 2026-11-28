# GaussianSet Data Structure

Also called: 3D Gaussian Representation, Gaussian Set

## Overview
The GaussianSet is a data structure representing a collection of 3D Gaussians, storing means, orientations, scales, opacities, and spherical harmonics coefficients, and computing covariance matrices [S1, unofficial]. It solves the problem of efficiently organizing explicit volumetric primitives for GPU batch processing and differentiable rasterization [S1, unofficial]. It sets itself apart from hierarchical scene representations by using a flat, contiguous array layout optimized for vectorized tensor operations [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Input**: A set of $N$ Gaussian primitives defined by their mean position, rotation, scale, opacity, and spherical harmonics coefficients [S1, unofficial].
*   **Output**: A structured data object containing contiguous tensors for all attributes, ready for projection and rasterization [S1, unofficial].

### Data Structures and Layout
The `GaussianSet` consists of the following fields, typically implemented as contiguous column-major tensors [S1, unofficial]:

1.  **`means`**: `Float32[N, 3]`
    *   The 3D position $\boldsymbol{\mu}$ of each Gaussian [S1, unofficial].
2.  **`quats`**: `Float32[N, 4]`
    *   The orientation represented as unit quaternions $q = (w, x, y, z)$ [S1, unofficial].
3.  **`scales`**: `Float32[N, 3]`
    *   The local scaling factors $s_x, s_y, s_z$ [S1, unofficial].
4.  **`opacities`**: `Float32[N]`
    *   The scalar opacity $\alpha$ for each Gaussian [S1, unofficial].
5.  **`sh_coefficients`**: `Float32[N, S, 3]` [S1, unofficial]
    *   Spherical Harmonics coefficients for view-dependent color, where $S$ is the number of SH bands [S1, unofficial].
6.  **`covariances`**: `Float32[N, 3, 3]`
    *   The precomputed $3 \times 3$ world-space covariance matrix $\Sigma$ for each Gaussian [S1, unofficial].

### Computation Steps
The `covariances` field is derived from `quats` and `scales` using the following steps [S1, unofficial]:

1.  **Normalize Quaternions**: Ensure $q$ is a unit quaternion [S1, unofficial].
    $$ q_{norm} = \frac{q}{\|q\|_2} $$
2.  **Convert Quaternion to Rotation Matrix**: Construct the $3 \times 3$ rotation matrix $R$ from $q_{norm}$ [G].
3.  **Construct Local Covariance Matrix**: Create the diagonal matrix $S$ from the scales [S1, unofficial].
    $$ S = \text{diag}(s_x^2, s_y^2, s_z^2) $$
4.  **Compute World Covariance**: Calculate the world-space covariance $\Sigma$ [S1, unofficial].
    $$ \Sigma = R S R^T $$

### Pseudocode
The following Python-like pseudocode demonstrates the construction of the `covariances` field from the raw attributes [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_covariances(quats: torch.Tensor, scales: torch.Tensor) -> torch.Tensor:
    """
    Computes the world-space covariance matrix for a set of Gaussians.
    
    Args:
        quats: Tensor of shape (N, 4), quaternions (w, x, y, z).
        scales: Tensor of shape (N, 3), local scaling factors.
        
    Returns:
        covariances: Tensor of shape (N, 3, 3), world-space covariance matrices.
    """
    # 1. Normalize quaternions to ensure unit length
    quats = quats / torch.norm(quats, dim=1, keepdim=True)
    
    # 2. Convert quaternions to rotation matrices
    def quat_to_rot(q):
        w, x, y, z = q[:, 0], q[:, 1], q[:, 2], q[:, 3]
        R = torch.zeros(q.shape[0], 3, 3, device=q.device)
        R[:, 0, 0] = 1 - 2 * (y**2 + z**2)
        R[:, 0, 1] = 2 * (x*y - w*z)
        R[:, 0, 2] = 2 * (x*z + w*y)
        R[:, 1, 0] = 2 * (x*y + w*z)
        R[:, 1, 1] = 1 - 2 * (x**2 + z**2)
        R[:, 1, 2] = 2 * (y*z - w*x)
        R[:, 2, 0] = 2 * (x*z - w*y)
        R[:, 2, 1] = 2 * (y*z + w*x)
        R[:, 2, 2] = 1 - 2 * (x**2 + y**2)
        return R

    R = quat_to_rot(quats)
    
    # 3. Construct local covariance matrix S (diagonal)
    scales_sq = scales ** 2
    S = torch.zeros_like(R)
    S[:, 0, 0] = scales_sq[:, 0]
    S[:, 1, 1] = scales_sq[:, 1]
    S[:, 2, 2] = scales_sq[:, 2]
    
    # 4. Compute world covariance: Cov = R * S * R^T
    covariances = torch.bmm(R, torch.bmm(S, R.transpose(1, 2)))
    
    return covariances
```

### Parameters and Edge Cases
*   **Quaternion Normalization**: Quaternions must be normalized before conversion to rotation matrices to ensure valid rotation [S1, unofficial].
*   **Scale Exponentiation**: Raw stored scales are exponentiated (`torch.exp`) to ensure they remain positive [S1, unofficial]. The `GaussianSet` may store the exponentiated values directly or compute them on the fly [S1, unofficial].
*   **Opacity Sigmoid**: Raw stored opacities are sigmoided (`torch.sigmoid`) to bound them between 0 and 1 [S1, unofficial].
*   **SH Concatenation**: Spherical harmonics coefficients from different degrees are concatenated along the band dimension [S1, unofficial].
*   **Precision**: Using `Float32` is standard, but `Float64` may be used for high-precision training to avoid gradient instability [G].

## Subtopics

### [ECoNGS Optimization and Coding](2087-econgs-optimization-and-coding.md)
A scene representation method that compresses Gaussian attributes via neural entropy coding to reduce storage and computational costs. It sets itself apart from standard explicit storage by using anchor attribute optimization and joint context models to exploit inter-scene redundancy. Choose it when memory footprint and transmission bandwidth are critical constraints.

### [GaussianList Data Structure and Initialization](2214-gaussianlist-data-structure-and-initialization.md)
A contiguous, column-major data structure designed for batch processing during differentiable volume rendering. It sets itself apart from hierarchical variants like Octree-GS by using a flat array, and provides the specific memory layout and initialization logic required by the rasterization pipeline. Choose it when maximizing GPU throughput via contiguous memory access is the priority.

### [Gaussian Property Loading](2221-gaussian-property-loading.md)
The method for ingesting 3D Gaussian Splatting scene data from PLY files and decoding raw vertex properties into the explicit mathematical parameters required by the rasterization pipeline. It sets itself apart by focusing strictly on initial ingestion and decoding, whereas sibling methods handle spatial alignment or legacy format variations. Choose it as the first step in the data ingestion pipeline.

### [3D Covariance Computation](2231-3d-covariance-computation.md)
The algorithm for calculating the $3 \times 3$ covariance matrix from rotation and scale parameters. It sets itself apart from direct covariance optimization by explicitly decoupling rotation and scale, ensuring the resulting matrix is always positive semi-definite. Choose it when physically interpretable, guaranteed-stable covariance matrices are required.

### [3D Gaussian Data Layout](2242-3d-gaussian-data-layout.md)
Defines the in-memory representation of a single 3D Gaussian primitive for differentiable rendering. It sets itself apart by explicitly separating the logical primitive definition from the physical memory layout (e.g., Structure of Arrays vs. Array of Structures) to facilitate efficient GPU batch processing. Choose it when optimizing memory access patterns for specific hardware architectures.

### [3D Gaussian Splatting (3DGS) Preliminaries](2257-3d-gaussian-splatting-3dgs-preliminaries.md)
The broader context of 3DGS as an explicit scene representation that replaces implicit neural fields with discrete, differentiable volumetric splats. It sets itself apart by enabling real-time rendering through GPU-friendly tile-based rasterization. Choose it as the foundational reference for the entire rendering pipeline.

### [N-DGS Bounded Cholesky Parameterization](2811-n-dgs-bounded-cholesky-parameterization.md)
A parameterization for the joint covariance of 3D Gaussians that uses bounded Cholesky coefficients multiplied by query diagonal scales. It sets itself apart from direct regression methods by implicitly deriving the regression operator from the ambient joint Cholesky factor. Choose it when modeling cross-covariance and spatial extent control is necessary.

### [Gaussian-based Representation](5918-gaussian-based-representation.md)
A general concept that encodes signals, data, or geometric structures as an explicit, continuous sum of parameterized Gaussian functions. It sets itself apart from Gaussian Mixture Models by approximating the signal value itself rather than modeling the data distribution. Choose it for compact, analytically tractable representations of high-dimensional data.

## Operating characteristics
*   **Memory Complexity**: Not given in the sources. [S1, unofficial]
*   **Time Complexity**: Not given in the sources. [S1, unofficial]
*   **Strengths**: Flat array layout allows for efficient GPU batch processing and vectorized operations [S1, unofficial]. Precomputed covariances avoid redundant calculations during rasterization [S1, unofficial].
*   **Limitations**: Storing all attributes explicitly can lead to high memory usage for large scenes [S1, unofficial]. Compression techniques like [ECoNGS](2087-econgs-optimization-and-coding.md) are required for scalability [S1, unofficial].
*   **When to Use**: Not given in the sources. [S1, unofficial]
*   **When Not to Use**: Not given in the sources. [S1, unofficial]

## Background
The origin and specific design history of this data structure are not given in the sources. [S1, unofficial] It is used within implementations of 3D Gaussian Splatting rendering pipelines [S1, unofficial].

## Sources
[S1] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
