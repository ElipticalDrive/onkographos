# 3D Covariance Computation

Also called: 3D Covariance Construction, Covariance Matrix Computation from Quaternions and Scales, Gaussian covariance from quaternion and scale

## Overview
Computes the 3D covariance matrix of a Gaussian from its scale and rotation parameters [S1, S2, S3, unofficial]. It solves the problem of representing the spatial extent and orientation of a 3D Gaussian in world coordinates by transforming local axis-aligned variances into a full symmetric positive-definite matrix [S2, S3, unofficial]. What sets it apart from direct covariance optimization is that it derives the covariance from unconstrained rotation (quaternion) and scale parameters, ensuring the resulting matrix is always valid without requiring projection steps [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - `quats`: Tensor of shape $(N, 4)$ containing unit quaternions representing orientations [S2, unofficial].
  - `scales`: Tensor of shape $(N, 3)$ containing scales along each axis, representing standard deviations in 3D [S2, unofficial].
- **Output:**
  - `cov3d`: Float32 tensor of shape $(N, 3, 3)$ containing the computed covariance matrices for each of the $N$ Gaussians [S2, unofficial].

**Data Structures:**
- The rotation matrix $R$ is a $3 \times 3$ matrix derived from the quaternion [S2, unofficial].
- The local scale matrix $S$ is a $3 \times 3$ diagonal matrix where diagonal elements are the squared scale values (variances) [S2, unofficial].

**Steps:**
1. **Convert Quaternions to Rotation Matrix:** Convert the input quaternions to a batch of $3 \times 3$ rotation matrices $R$ [S1, S2, S3, unofficial].
2. **Construct Local Scale Matrix:** Create a diagonal matrix $S$ where the diagonal elements are the squared scale values [S1, S2, unofficial].
3. **Compute Covariance:** Compute the covariance matrix using the similarity transformation $\Sigma = R S R^T$ [S2, S3, unofficial].
4. **Store Result:** Store the result in the Gaussian set's covariance buffer [S1, unofficial].

**Formulas:**
The covariance matrix $\Sigma$ is defined as:
$$ \Sigma = R \cdot S \cdot R^T $$
where:
- $R$ is the rotation matrix derived from the quaternion [S2, unofficial].
- $S = \text{diag}(s_x^2, s_y^2, s_z^2)$ is the local covariance matrix with variances along each axis [S2, unofficial].
- $s_x, s_y, s_z$ are the scale components [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_cov3d(quats: torch.Tensor, scales: torch.Tensor) -> torch.Tensor:
    """
    Computes the 3D covariance matrix from quaternions and scales.
    
    Args:
        quats: Tensor of shape (N, 4) containing unit quaternions.
        scales: Tensor of shape (N, 3) containing scales along each axis.
        
    Returns:
        cov3d: Tensor of shape (N, 3, 3) containing the covariance matrices.
    """
    # Step 1: Convert quaternions to rotation matrices
    # Assuming a standard quaternion_to_rotation_matrix function exists
    # or implementing the conversion manually.
    # Here we assume a helper function is available or implemented.
    R = quaternion_to_rotation_matrix(quats)
    
    # Step 2: Construct local scale matrix S
    # S is a diagonal matrix with squared scales
    # S = diag(s_x^2, s_y^2, s_z^2)
    scales_sq = scales ** 2  # Shape (N, 3)
    
    # Create a zero tensor for S
    S = torch.zeros_like(scales_sq).unsqueeze(1).expand(-1, 3, -1) # Shape (N, 3, 3)
    S[:, 0, 0] = scales_sq[:, 0]
    S[:, 1, 1] = scales_sq[:, 1]
    S[:, 2, 2] = scales_sq[:, 2]
    
    # Step 3: Compute covariance matrix using similarity transformation
    # cov3d = R @ S @ R^T
    # Using einsum for batched matrix multiplication
    cov3d = torch.einsum("nij,njk,nlk->nil", R, S, R)
    
    return cov3d

def quaternion_to_rotation_matrix(quats: torch.Tensor) -> torch.Tensor:
    """
    Converts a batch of quaternions to rotation matrices.
    
    Args:
        quats: Tensor of shape (N, 4) containing unit quaternions [w, x, y, z].
        
    Returns:
        R: Tensor of shape (N, 3, 3) containing rotation matrices.
    """
    # Normalize quaternions to ensure they are unit quaternions
    quats = quats / quats.norm(dim=1, keepdim=True)
    
    w, x, y, z = quats[:, 0], quats[:, 1], quats[:, 2], quats[:, 3]
    
    # Compute rotation matrix elements
    # R = [
    #   [1 - 2(y^2 + z^2), 2(xy - wz),     2(xz + wy)],
    #   [2(xy + wz),       1 - 2(x^2 + z^2), 2(yz - wx)],
    #   [2(xz - wy),       2(yz + wx),     1 - 2(x^2 + y^2)]
    # ]
    
    R = torch.zeros((quats.shape[0], 3, 3), dtype=quats.dtype, device=quats.device)
    
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
```

**Edge Cases and Pitfalls:**
- **Precision:** Ensure quaternions are normalized before conversion to avoid rotation matrix errors [G].
- **Boundaries:** The method assumes valid input quaternions and non-negative scales [G].
- **Degenerate Input:** If scales are zero, the covariance matrix will be zero, which may cause issues in rendering [G].

## Subtopics

### [Pre-computed Covariance and Color Buffer Generation](2817-pre-computed-covariance-and-color-buffer-generation.md)
This method is a preprocessing step that converts raw Gaussian splat parameters (position, scale, rotation, color) into pre-computed 3x3 covariance matrices and normalized float colors to accelerate rendering. It solves the problem of per-frame matrix multiplication and color decoding by computing these values once during load, thereby reducing GPU compute overhead during the rasterization phase. It builds on [3D Covariance Computation](2231-3d-covariance-computation.md) by providing the final symmetric covariance representation required for the splatting shader.

### [Covariance Factorisation](2894-covariance-factorisation.md)
Covariance Factorisation parameterizes the 3D Gaussian covariance matrix by decomposing it into unconstrained rotation and scale components, ensuring the resulting matrix remains positive semi-definite (PSD) during gradient-based optimization. This approach distinguishes itself from direct covariance optimization by decoupling geometric shape (scale) from orientation (rotation), allowing stable unconstrained gradient descent without requiring projection steps to enforce PSD constraints.

## Operating characteristics
- **Time Complexity:** $O(N)$ for $N$ Gaussians, as each Gaussian's covariance is computed independently.
- **Memory Complexity:** $O(N)$ to store the resulting covariance matrices.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of Gaussians.
- **Strengths:** Simple and efficient computation; ensures valid covariance matrices.
- **Limitations:** Requires quaternion to rotation matrix conversion, which can be computationally expensive for large $N$ [G].
- **When to Use:** When explicit covariance matrices are needed for rendering or optimization.
- **When Not to Use:** When memory is constrained and covariance matrices can be computed on-the-fly.

## Background
Originated in the context of 3D Gaussian Splatting for novel view synthesis [S1, S2, S3, unofficial]. The method is a fundamental component of the GaussianSet Data Structure, which stores means, orientations, scales, opacities, and spherical harmonics coefficients [Parent]. It is used in various implementations of 3D Gaussian Splatting, including [thomasantony/splat](https://github.com/thomasantony/splat) [S1, unofficial], [sascha-kirch/splaty](https://github.com/sascha-kirch/splaty) [S2, unofficial], and [Fhz2346/3D-Gaussian-Splatting](https://github.com/Fhz2346/3D-Gaussian-Splatting) [S3, unofficial].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] sascha-kirch/splaty, https://github.com/sascha-kirch/splaty, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
