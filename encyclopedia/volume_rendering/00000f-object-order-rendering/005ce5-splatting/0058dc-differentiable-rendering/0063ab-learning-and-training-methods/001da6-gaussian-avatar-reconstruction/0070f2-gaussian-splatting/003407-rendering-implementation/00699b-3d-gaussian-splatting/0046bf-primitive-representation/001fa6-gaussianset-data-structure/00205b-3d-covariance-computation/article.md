# 3D Covariance Computation

Also called: 3D Covariance Construction, Covariance Matrix Computation from Quaternions and Scales, Gaussian Splatting Covariance Computation, Gaussian covariance from quaternion and scale

## Overview
3D Covariance Computation calculates the $3 \times 3$ covariance matrix $\boldsymbol{\Sigma}$ for a set of 3D Gaussians from their stored rotation (quaternion) and scale parameters [S1, S2, S3, S4, unofficial]. It solves the problem of representing the spatial extent and orientation of each Gaussian in world coordinates, which is required for projecting the Gaussian to 2D screen space for rendering [S3, unofficial]. It sets itself apart from direct optimization of covariance entries by explicitly decoupling rotation and scale into distinct learnable components, ensuring the resulting matrix is always positive semi-definite (PSD) and physically interpretable [S1, S3, unofficial].

## How it works
**Inputs:**
- `quats`: Tensor of shape $(N, 4)$ containing unit quaternions $(w, x, y, z)$ representing orientations [S2, S4, unofficial].
- `scales`: Tensor of shape $(N, 3)$ containing scale factors $(s_x, s_y, s_z)$ along each axis, representing standard deviations in 3D [S2, S4, unofficial].

**Outputs:**
- `covariances`: Float32 tensor of shape $(N, 3, 3)$ containing the computed covariance matrices $\boldsymbol{\Sigma}$ for each of the $N$ Gaussians [S2, unofficial].

**Data Structures:**
- The rotation matrix $\mathbf{R}$ is a $3 \times 3$ matrix derived from the quaternion.
- The local scale matrix $\mathbf{S}$ is a $3 \times 3$ diagonal matrix containing the squared scales (variances).

**Steps:**
1. **Normalize Quaternions:** Ensure input quaternions are unit length. If $q = (w, x, y, z)$, compute $q_{norm} = q / \|q\|$ [S4, unofficial].
2. **Construct Rotation Matrix:** Convert the normalized quaternion to a $3 \times 3$ rotation matrix $\mathbf{R}$ [S1, S2, S3, S4, unofficial].
3. **Construct Scale Matrix:** Create a diagonal matrix $\mathbf{S}$ where the diagonal elements are the squared scale values: $S_{ii} = s_i^2$ [S1, S2, unofficial].
4. **Compute Covariance:** Calculate the world-space covariance matrix using the similarity transformation $\boldsymbol{\Sigma} = \mathbf{R} \mathbf{S} \mathbf{R}^T$ [S2, S3, unofficial]. Alternatively, compute $\mathbf{L} = \mathbf{R} \cdot \text{diag}(\mathbf{s})$ and then $\boldsymbol{\Sigma} = \mathbf{L} \mathbf{L}^T$ [S4, unofficial].

**Formulas:**
The covariance matrix $\boldsymbol{\Sigma}$ is defined as:
$$ \boldsymbol{\Sigma} = \mathbf{R} \mathbf{S} \mathbf{R}^T $$
where:
- $\mathbf{R}$ is the rotation matrix derived from the quaternion.
- $\mathbf{S} = \text{diag}(s_x^2, s_y^2, s_z^2)$ is the local covariance matrix with variances along each axis [S2, S3, unofficial].

The rotation matrix $\mathbf{R}$ from a unit quaternion $q = w + xi + yj + zk$ is:
$$ \mathbf{R} = \begin{bmatrix} 1 - 2(y^2 + z^2) & 2(xy - wz) & 2(xz + wy) \\ 2(xy + wz) & 1 - 2(x^2 + z^2) & 2(yz - wx) \\ 2(xz - wy) & 2(yz + wx) & 1 - 2(x^2 + y^2) \end{bmatrix} $$
[Derived from standard quaternion-to-matrix conversion, consistent with [S1, S2, S3, S4, unofficial]].

**Parameters:**
- `quats`: Input tensor of shape $(N, 4)$ [S2, S4, unofficial].
- `scales`: Input tensor of shape $(N, 3)$ [S2, S4, unofficial].

**Edge Cases and Pitfalls:**
- **Non-unit Quaternions:** Quaternions must be normalized before conversion to ensure a valid rotation matrix [S4, unofficial].
- **Isotropic Scales:** If scales are isotropic (single scalar), the scale matrix is constructed from that scalar repeated on the diagonal [S3, unofficial].
- **Numerical Stability:** The decomposition $\boldsymbol{\Sigma} = \mathbf{R} \mathbf{S} \mathbf{R}^T$ guarantees $\boldsymbol{\Sigma}$ is positive semi-definite, avoiding numerical instability during gradient-based training [S1, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def compute_cov3d(quats: torch.Tensor, scales: torch.Tensor) -> torch.Tensor:
    """
    Computes the 3D covariance matrix for a set of Gaussians.
    
    Args:
        quats: Tensor of shape (N, 4) containing quaternions (w, x, y, z).
        scales: Tensor of shape (N, 3) containing scale factors (sx, sy, sz).
        
    Returns:
        covariances: Tensor of shape (N, 3, 3) containing covariance matrices.
    """
    # Step 1: Normalize quaternions
    quats = quats / torch.norm(quats, dim=1, keepdim=True)
    
    # Extract components
    w = quats[:, 0]
    x = quats[:, 1]
    y = quats[:, 2]
    z = quats[:, 3]
    
    # Step 2: Construct rotation matrix R
    # R is 3x3 for each Gaussian
    # Using batched operations
    R = torch.zeros(quats.shape[0], 3, 3, device=quats.device, dtype=quats.dtype)
    
    R[:, 0, 0] = 1 - 2 * (y**2 + z**2)
    R[:, 0, 1] = 2 * (x*y - w*z)
    R[:, 0, 2] = 2 * (x*z + w*y)
    
    R[:, 1, 0] = 2 * (x*y + w*z)
    R[:, 1, 1] = 1 - 2 * (x**2 + z**2)
    R[:, 1, 2] = 2 * (y*z - w*x)
    
    R[:, 2, 0] = 2 * (x*z - w*y)
    R[:, 2, 1] = 2 * (y*z + w*x)
    R[:, 2, 2] = 1 - 2 * (x**2 + y**2)
    
    # Step 3: Construct scale matrix S (diagonal)
    # S = diag(sx^2, sy^2, sz^2)
    scales_sq = scales**2
    S = torch.zeros(quats.shape[0], 3, 3, device=quats.device, dtype=quats.dtype)
    S[:, 0, 0] = scales_sq[:, 0]
    S[:, 1, 1] = scales_sq[:, 1]
    S[:, 2, 2] = scales_sq[:, 2]
    
    # Step 4: Compute covariance Sigma = R @ S @ R^T
    # Using einsum for efficiency: cov[i,j,k] = sum_n sum_m R[i,j,n] * S[n,m] * R[k,m]
    # Or simply: L = R @ diag(s), then Cov = L @ L^T
    # Let's use the L @ L^T approach as it is numerically stable and common
    # L = R @ diag(s)
    # L[i,j,k] = sum_n R[i,j,n] * diag(s)[n,k] = R[i,j,k] * s[k] (since diag(s) is diagonal)
    L = R * scales.unsqueeze(1) # Broadcast scales (N,3) to (N,1,3)
    
    # Cov = L @ L^T
    covariances = torch.bmm(L, L.transpose(1, 2))
    
    return covariances
```

## Subtopics

### [Pre-computed Covariance and Color Buffer Generation](2817-pre-computed-covariance-and-color-buffer-generation.md)
This method is a preprocessing step that converts raw Gaussian splat parameters (position, scale, rotation, color) into pre-computed $3 \times 3$ covariance matrices and normalized float colors to accelerate rendering. It solves the problem of per-frame matrix multiplication and color decoding by computing these values once during load.

### [Covariance Factorisation](2894-covariance-factorisation.md)
Covariance Factorisation is the parameterization strategy used in 3D Gaussian Splatting to represent the 3D Gaussian covariance matrix $\boldsymbol{\Sigma}$ as a product of a rotation matrix $\mathbf{R}$ and a scaling matrix $\mathbf{S}$. It solves the problem that directly optimizing the entries of $\boldsymbol{\Sigma}$ can result in non-positive semi-definite (PSD) matrices, which violates physical interpretation and causes numerical instability during gradient-based training. It sets itself apart from sibling methods like [Cholesky Parameterisation for Gaussian Splats](3120-cholesky-parameterisation-for-gaussian-splats.md) by explicitly decoupling rotation and scale into distinct learnable components, rather than relying on a triangular factorization to implicitly encode orientation.

### [Automatic covariance and rotation recalculation](12554-automatic-covariance-and-rotation-recalculation.md)
This method updates the parameters of 3D Gaussians when the underlying mesh vertices are transformed, ensuring that splats correctly reflect the new geometry without requiring manual recalculation of scaling factors. It sets itself apart from vanilla Gaussian Splatting modifications, which often require explicit matrix updates that can introduce holes or blurriness, by moving vertices directly in $\mathbb{R}^3$ and automatically deriving the new covariance and rotation parameters.

### Gsplat to CovSplat Conversion
To convert a `Gsplat` to a `CovSplat`: 1. Initialize `covsplat.flags` to 0u. 2. Check if `isGsplatActive(gsplat.flags)` is true. 3. If active, copy `flags`, `index`, `rgba`, and `center` from `gsplat`. 4. Compute rotation matrix `m` by calling `scaleQuaternionToMatrix(gsplat.scales, gsplat.quaternion)`. 5. Compute the symmetric covariance matrix components: `m = m * transpose(m)`. 6. Extract diagonal elements into `covsplat.xxyyzz = vec3(m[0][0], m[1][1], m[2][2])`. 7. Extract off-diagonal elements into `covsplat.xyxzyz = vec3(m[0][1], m[0][2], m[1][2])` [S5, unofficial]. If the input `gsplat` is null or inactive, the output `covsplat.flags` remains 0u, and other fields are not updated (or initialized to defaults in the JS wrapper) [S5, unofficial].

### Anisotropic Covariance Optimization
Optimize full covariance matrix for 3D Gaussians to allow anisotropy [S6]. Vs Isotropic: Anisotropy significantly improves alignment with surfaces, allowing higher rendering quality while maintaining same point count. Isotropic optimization uses a single scalar value controlling radius on all three axes [S6].

## Operating characteristics
- **Time Complexity:** $O(N)$ for $N$ Gaussians, as each Gaussian's covariance is computed independently using constant-time matrix operations ($3 \times 3$).
- **Memory Complexity:** $O(N)$ to store the resulting $N$ covariance matrices.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of Gaussians.
- **Strengths:** Ensures PSD property, differentiable, efficient batched computation.
- **Limitations:** Requires quaternion normalization.
- **When to use:** When rendering 3D Gaussians in world space.
- **When not:** When using pre-computed covariances or isotropic approximations.

## Background
Origin: 3D Gaussian Splatting for Real-Time Radiance Field Rendering [S6]. History and development: The method was introduced in 2023 as part of the 3D Gaussian Splatting framework [S6]. Where it is used: Rendering of 3D Gaussian splats in real-time radiance field rendering [S6].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S4] xmed-lab/DIF-Gaussian, https://github.com/xmed-lab/DIF-Gaussian, fetched 2026-10-27 (unofficial)
[S5] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
[S6] 3D Gaussian Splatting for Real-Time Radiance Field Rendering, https://arxiv.org/abs/2308.04079, fetched 2026-10-26
