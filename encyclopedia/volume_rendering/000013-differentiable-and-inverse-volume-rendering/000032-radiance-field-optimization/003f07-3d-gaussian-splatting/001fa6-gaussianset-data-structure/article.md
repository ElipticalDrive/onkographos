# GaussianSet Data Structure

Also called: 3D Gaussian Representation, Gaussian Set

## Overview
The GaussianSet is the primary container for the explicit scene representation in 3D Gaussian Splatting, storing the parametric attributes of millions of anisotropic 3D Gaussian primitives [S1, unofficial]. It solves the problem of efficiently managing and computing with large sets of spatial primitives by organizing means, orientations, scales, opacities, and spherical harmonics (SH) coefficients into fixed-layout tensors [S1, unofficial]. It sets itself apart from the [GaussianList Data Structure](2214-gaussianlist-data-structure-and-initialization.md) by providing the high-level logical grouping and covariance computation logic, whereas the GaussianList exposes the raw memory layout for low-level batch processing [S1, unofficial].

## How it works
**Inputs and Outputs:**
The GaussianSet is constructed from optimized parameters (typically loaded from a checkpoint or PLY file) and outputs a structured object containing raw attributes and pre-computed world-space covariance matrices for rendering.

**Data Structures:**
The data is stored in PyTorch tensors with the following exact layouts [S1, unofficial]:
*   `means`: `Float32 Tensor, (N, 3)` — World coordinates of the Gaussian centers.
*   `quats`: `Float32 Tensor, (N, 4)` — Unit quaternions representing orientation.
*   `scales`: `Float32 Tensor, (N, 3)` — Log-space scales (exponentiated to get actual scale).
*   `opacities`: `Float32 Tensor, (N)` — Log-space opacities (sigmoided to get actual opacity).
*   `sh_coefficients`: `Float32 Tensor, (N, S, 3)` — Spherical Harmonics coefficients, where $S$ is the number of SH bands.
*   `covariances`: `Float32 Tensor, (N, 3, 3)` — Pre-computed world-space covariance matrices.

**Numbered Steps:**
1.  **Load Raw Parameters:** Retrieve raw tensors from the source (e.g., checkpoint or PLY).
2.  **Decode Attributes:** Apply inverse activation functions to recover physical values:
    *   Normalize quaternions to unit length.
    *   Exponentiate scales.
    *   Sigmoid opacities.
    *   Concatenate SH bands if stored separately.
3.  **Compute Covariance:** For each Gaussian $i$, compute the world-space covariance matrix $\Sigma_i$ using the rotation matrix $R_i$ derived from the quaternion and the diagonal scale matrix $S_i$.
4.  **Store:** Assemble the decoded attributes and computed covariances into the `GaussianSet` object.

**Formulas:**
The local covariance matrix $S$ is diagonal with variances derived from the scales $sx, sy, sz$ [S1, unofficial]:
$$ S = \text{diag}(sx^2, sy^2, sz^2) $$
The world-space covariance matrix $Cov$ is computed as [S1, unofficial]:
$$ Cov = R S R^T $$
where $R$ is the $3 \times 3$ rotation matrix constructed from the normalized quaternion.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def normalize_quaternions(quats):
    """Normalize quaternions to unit length."""
    return quats / torch.norm(quats, dim=-1, keepdim=True)

def compute_rotation_matrix(quat):
    """Convert a unit quaternion (w, x, y, z) to a 3x3 rotation matrix."""
    w, x, y, z = quat[:, 0], quat[:, 1], quat[:, 2], quat[:, 3]
    # Standard quaternion-to-matrix conversion
    R = torch.zeros((quat.shape[0], 3, 3), device=quat.device)
    R[:, 0, 0] = 1 - 2 * (y**2 + z**2)
    R[:, 0, 1] = 2 * (x * y - w * z)
    R[:, 0, 2] = 2 * (x * z + w * y)
    R[:, 1, 0] = 2 * (x * y + w * z)
    R[:, 1, 1] = 1 - 2 * (x**2 + z**2)
    R[:, 1, 2] = 2 * (y * z - w * x)
    R[:, 2, 0] = 2 * (x * z - w * y)
    R[:, 2, 1] = 2 * (y * z + w * x)
    R[:, 2, 2] = 1 - 2 * (x**2 + y**2)
    return R

def decode_gaussian_set(raw_params):
    """
    Decodes raw parameters from a checkpoint/PLY into a GaussianSet.
    
    Args:
        raw_params: Dict containing 'means', 'quats', 'scales', 'opacities', 'sh0', 'shN'.
        
    Returns:
        A dictionary representing the GaussianSet with decoded attributes and covariances.
    """
    # 1. Load and Decode Attributes
    means = raw_params['means']
    
    # Normalize quaternions
    quats = normalize_quaternions(raw_params['quats'])
    
    # Exponentiate scales
    scales = torch.exp(raw_params['scales'])
    
    # Sigmoid opacities
    opacities = torch.sigmoid(raw_params['opacities'])
    
    # Concatenate SH coefficients
    sh_coefficients = torch.cat([raw_params['sh0'], raw_params['shN']], dim=-2)
    
    # 2. Compute Covariances
    # Compute rotation matrices from quaternions
    R = compute_rotation_matrix(quats)
    
    # Compute local scale matrices (diagonal)
    # scales shape: (N, 3) -> square element-wise -> (N, 3)
    scales_sq = scales ** 2
    # Construct diagonal matrix for each Gaussian
    # S_i = diag(sx^2, sy^2, sz^2)
    # We can use einsum or manual construction. Manual for clarity:
    # Cov = R @ S @ R^T
    # S is diagonal, so R @ S is just scaling columns of R
    # R_scaled[:, :, 0] = R[:, :, 0] * scales_sq[:, 0]
    # R_scaled[:, :, 1] = R[:, :, 1] * scales_sq[:, 1]
    # R_scaled[:, :, 2] = R[:, :, 2] * scales_sq[:, 2]
    
    R_scaled = R * scales_sq[:, None, :]  # Broadcast scale to columns
    
    # Cov = R_scaled @ R^T
    covariances = torch.bmm(R_scaled, R.transpose(1, 2))
    
    # 3. Assemble Set
    gaussian_set = {
        'means': means,
        'quats': quats,
        'scales': scales,
        'opacities': opacities,
        'sh_coefficients': sh_coefficients,
        'covariances': covariances
    }
    
    return gaussian_set
```

**Edge Cases and Pitfalls:**
*   **Quaternion Normalization:** Quaternions must be normalized before rotation matrix construction to ensure valid rotation [S1, unofficial].
*   **Scale Exponentiation:** Raw stored scales are in log-space; failing to exponentiate results in incorrect spatial extent [S1, unofficial].
*   **Opacity Sigmoid:** Raw stored opacities are in logit-space; failing to sigmoid results in invalid opacity values [S1, unofficial].
*   **SH Concatenation:** SH coefficients may be stored in separate bands (e.g., degree 0 and degree 1); they must be concatenated along the coefficient dimension before rendering [S1, unofficial].

## Subtopics

### [ECoNGS Optimization and Coding](2087-econgs-optimization-and-coding.md)
Compresses Gaussian attributes using neural entropy coding to reduce storage and computational costs. Choose when storage efficiency is critical.

### [GaussianList Data Structure and Initialization](2214-gaussianlist-data-structure-and-initialization.md)
Provides the contiguous, column-major memory layout for low-level batch processing. Choose when implementing the core rasterizer kernel.

### [Gaussian Property Loading](2221-gaussian-property-loading.md)
Parses PLY files into the GaussianSet structure. Choose when ingesting scene data from disk.

### [3D Covariance Computation](2231-3d-covariance-computation.md)
Details the mathematical derivation of the covariance matrix from quaternion and scale. Choose when implementing the projection step.

### [3D Gaussian Data Layout](2242-3d-gaussian-data-layout.md)
Defines the internal fields of a single Gaussian primitive. Choose when designing the primitive class.

### [3D Gaussian Splatting (3DGS) Preliminaries](2257-3d-gaussian-splatting-3dgs-preliminaries.md)
The foundational method using fixed-degree SH and heuristic density control. Choose as the baseline implementation.

### [N-DGS Bounded Cholesky Parameterization](2811-n-dgs-bounded-cholesky-parameterization.md)
Uses bounded Cholesky coefficients for stable covariance parameterization. Choose when stability of joint covariance is required.

### [Gaussian-based Representation](5918-gaussian-based-representation.md)
General mathematical formalism for Gaussian superposition. Choose for theoretical understanding or non-rendering applications.

## Operating characteristics
*   **Time Complexity:** Covariance computation is $O(N)$ where $N$ is the number of Gaussians, involving matrix multiplications per Gaussian.
*   **Memory Complexity:** $O(N \cdot (3 + 4 + 3 + 1 + S \cdot 3 + 9))$ floats, where $S$ is the number of SH bands. For standard 3DGS ($S=9$ for degree 1, or more for higher degrees), this is significant.
*   **Strengths:** Explicit representation allows for fast GPU rasterization without ray marching.
*   **Limitations:** Large memory footprint for high-resolution scenes with many Gaussians.
*   **When to use:** Real-time novel view synthesis where rendering speed is paramount.
*   **When not to use:** Applications requiring extreme compression or implicit scene representation.

## Background
The GaussianSet is a core component of 3D Gaussian Splatting, introduced by Kerbl et al. in 2023 [S1, unofficial]. It replaces implicit neural fields with an explicit set of anisotropic 3D Gaussian primitives, enabling real-time rendering via GPU rasterization [S1, S2, S5, S7, S9, unofficial]. The data structure evolved to support the differentiable optimization of geometric and photometric attributes [S9, S12, unofficial].

## Sources
[S1] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
