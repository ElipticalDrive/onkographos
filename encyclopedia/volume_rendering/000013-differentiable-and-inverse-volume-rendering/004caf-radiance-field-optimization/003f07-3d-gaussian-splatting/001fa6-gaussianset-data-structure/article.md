# GaussianSet Data Structure

Also called: 3D Gaussian Representation, Gaussian Set

## Overview
The GaussianSet is a class representing a collection of 3D Gaussian primitives, storing their geometric and photometric attributes (means, orientations, scales, opacities, and spherical harmonics coefficients) and providing methods to compute their spatial covariance matrices [S1, unofficial]. It serves as the primary in-memory container for differentiable volume rendering, bridging the gap between raw scene data and the rasterization pipeline by organizing attributes for batch processing and ensuring mathematical consistency through normalization and activation functions [S1, unofficial]. It differs from sibling structures like [GaussianList](2214-gaussianlist-data-structure-and-initialization.md) by exposing the high-level logical grouping and initialization logic, whereas [3D Gaussian Data Layout](2242-3d-gaussian-data-layout.md) defines the contiguous memory footprint of individual primitives.

## How it works
**Inputs and Outputs:**
*   **Input:** Raw checkpoint data or PLY file attributes containing unnormalized quaternions, log-scaled scales, linear opacities, and split spherical harmonics coefficients.
*   **Output:** A normalized `GaussianSet` object with valid, positive-definite covariance matrices and physically plausible opacity values, ready for projection and rendering.

**Data Structures:**
The `GaussianSet` maintains the following fields, where $N$ is the number of Gaussians and $S$ is the number of SH bands per color channel:
*   `means`: `Float32 Tensor` of shape $(N, 3)$. World-space coordinates.
*   `quats`: `Float32 Tensor` of shape $(N, 4)$. Unnormalized rotation quaternions.
*   `scales`: `Float32 Tensor` of shape $(N, 3)$. Log-space scale factors.
*   `opacities`: `Float32 Tensor` of shape $(N,)$. Linear opacity values.
*   `sh_coefficients`: `Float32 Tensor` of shape $(N, S, 3)$. Concatenated spherical harmonics coefficients.
*   `covariances`: `Float32 Tensor` of shape $(N, 3, 3)$. Precomputed world-space covariance matrices.

**Numbered Steps:**
1.  **Normalize Quaternions:** The raw quaternion data is normalized to unit length to ensure valid rotation representations [S1, unofficial].
    $$ \mathbf{q}_{norm} = \frac{\mathbf{q}}{\|\mathbf{q}\|_2} $$
2.  **Exponentiate Scales:** The raw scale values, stored in log-space, are exponentiated to obtain positive physical scales [S1, unofficial].
    $$ \mathbf{s} = \exp(\mathbf{s}_{raw}) $$
3.  **Sigmoid Opacities:** The raw opacity values are passed through a sigmoid function to constrain them to the range $(0, 1)$ [S1, unofficial].
    $$ \alpha = \sigma(\alpha_{raw}) = \frac{1}{1 + e^{-\alpha_{raw}}} $$
4.  **Concatenate SH Coefficients:** If the input data splits spherical harmonics into a base level (degree 0) and higher levels, they are concatenated along the band dimension [S1, unofficial].
    $$ \mathbf{C}_{SH} = \text{concat}(\mathbf{C}_{SH0}, \mathbf{C}_{SHN}) $$
5.  **Compute Covariance Matrices:** For each Gaussian, compute the world-space covariance matrix $\mathbf{\Sigma}$ using the rotation matrix $\mathbf{R}$ derived from the normalized quaternion and the diagonal scale matrix $\mathbf{S}$ derived from the exponentiated scales [S1, unofficial].
    $$ \mathbf{\Sigma} = \mathbf{R} \mathbf{S} \mathbf{R}^T $$
    Where $\mathbf{S} = \text{diag}(s_x^2, s_y^2, s_z^2)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def normalize_quaternions(quats: torch.Tensor) -> torch.Tensor:
    """Normalize quaternions to unit length."""
    return torch.nn.functional.normalize(quats, p=2, dim=-1)

def exponentiate_scales(scales_raw: torch.Tensor) -> torch.Tensor:
    """Exponentiate log-scale values."""
    return torch.exp(scales_raw)

def sigmoid_opacities(opacities_raw: torch.Tensor) -> torch.Tensor:
    """Apply sigmoid to constrain opacities to (0, 1)."""
    return torch.sigmoid(opacities_raw)

def quaternion_to_rotation_matrix(quats: torch.Tensor) -> torch.Tensor:
    """Convert normalized quaternions (N, 4) to rotation matrices (N, 3, 3).
    Assumes w, x, y, z order or handles standard conversion.
    """
    # Standard conversion from quaternion to rotation matrix
    # q = [w, x, y, z]
    w, x, y, z = torch.unbind(quats, dim=-1)
    R = torch.zeros(quats.shape[0], 3, 3, device=quats.device)
    
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

def compute_covariances(quats: torch.Tensor, scales: torch.Tensor) -> torch.Tensor:
    """Compute world-space covariance matrices from quaternions and scales.
    
    Args:
        quats: Normalized quaternions (N, 4).
        scales: Exponentiated scales (N, 3).
        
    Returns:
        Covariance matrices (N, 3, 3).
    """
    R = quaternion_to_rotation_matrix(quats)
    # S is diagonal with variances s_x^2, s_y^2, s_z^2
    S_diag = scales**2
    S = torch.zeros_like(R)
    S[:, 0, 0] = S_diag[:, 0]
    S[:, 1, 1] = S_diag[:, 1]
    S[:, 2, 2] = S_diag[:, 2]
    
    # Cov = R * S * R^T
    Cov = torch.bmm(R, torch.bmm(S, R.transpose(1, 2)))
    return Cov

class GaussianSet:
    def __init__(self, means, quats_raw, scales_raw, opacities_raw, sh_coefficients):
        self.means = means
        self.quats = normalize_quaternions(quats_raw)
        self.scales = exponentiate_scales(scales_raw)
        self.opacities = sigmoid_opacities(opacities_raw)
        self.sh_coefficients = sh_coefficients
        self.covariances = compute_covariances(self.quats, self.scales)
```

**Parameters and Edge Cases:**
*   **Quaternion Order:** The pseudocode assumes a specific quaternion-to-matrix conversion. If the source data uses a different order (e.g., x, y, z, w), the conversion must be adjusted [G].
*   **Degenerate Scales:** If any scale value is zero or negative before exponentiation, the resulting covariance matrix will be singular or invalid. The exponentiation step ensures positivity [S1, unofficial].
*   **Precision:** Floating-point errors in quaternion normalization can lead to slight non-orthogonality in the rotation matrix. Re-normalization after conversion is not specified in the sources but may be required for numerical stability [G].

## Subtopics

### [ECoNGS Optimization and Coding](2087-econgs-optimization-and-coding.md)
Compresses Gaussian attributes using neural entropy coding to reduce storage and computational costs. Choose when memory bandwidth is a bottleneck.

### [GaussianList Data Structure and Initialization](2214-gaussianlist-data-structure-and-initialization.md)
Defines the contiguous, column-major memory layout for batch processing. Choose when implementing low-level GPU kernels.

### [Gaussian Property Loading](2221-gaussian-property-loading.md)
Parses PLY files into the GaussianSet structure. Choose when ingesting scene data from disk.

### [3D Covariance Computation](2231-3d-covariance-computation.md)
Details the derivation of the covariance matrix from quaternion and scale parameters. Choose when implementing the projection step.

### [3D Gaussian Data Layout](2242-3d-gaussian-data-layout.md)
Defines the in-memory representation of a single Gaussian primitive. Choose when optimizing cache locality.

### [3D Gaussian Splatting (3DGS) Preliminaries](2257-3d-gaussian-splatting-3dgs-preliminaries.md)
Provides the theoretical background for 3DGS. Choose for understanding the rendering pipeline context.

### [N-DGS Bounded Cholesky Parameterization](2811-n-dgs-bounded-cholesky-parameterization.md)
Uses bounded Cholesky coefficients for joint covariance control. Choose when modeling cross-covariance.

### [Gaussian-based Representation](5918-gaussian-based-representation.md)
General mathematical formalism for Gaussian superposition. Choose for theoretical analysis of signal approximation.

## Operating characteristics
*   **Time Complexity:** Covariance computation is $O(N)$ for $N$ Gaussians, with a constant factor of $3 \times 3$ matrix multiplications. Normalization and activation functions are also $O(N)$.
*   **Memory Complexity:** The structure stores $N \times (3 + 4 + 3 + 1 + 3S)$ floats, where $S$ is the number of SH bands. For typical SH degree 3 ($S=12$), this is approximately $3 \times 10 \times N$ bytes plus overhead.
*   **Strengths:** Explicit representation allows for efficient rasterization and differentiable optimization. The separation of logical attributes (GaussianSet) from memory layout (GaussianList) allows for flexible implementation.
*   **Limitations:** The structure does not handle spatial indexing (e.g., BVH or grid) for culling, which is typically handled by the rendering pipeline [S1, unofficial].
*   **When to Use:** Use when implementing the core scene representation for 3D Gaussian Splatting. Do not use if you need to optimize for extreme compression (use [ECoNGS](2087-econgs-optimization-and-coding.md)) or low-level memory layout (use [GaussianList](2214-gaussianlist-data-structure-and-initialization.md)).

## Background
The GaussianSet data structure originates from the 3D Gaussian Splatting (3DGS) method, which replaces implicit neural fields with explicit anisotropic 3D Gaussian primitives [S1, S2, S5, S7, S9, S12, unofficial]. It was developed to enable real-time rendering via GPU rasterization without ray marching, solving the trade-off between the high visual quality of NeRFs and the speed of explicit geometry [S5, S6, S8, S9, S12, unofficial]. The structure stores the parameters necessary to define each Gaussian primitive, including position, orientation, scale, opacity, and color, allowing for end-to-end optimization of geometric and photometric attributes [S9, S12, unofficial].

## Sources
[S1] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
