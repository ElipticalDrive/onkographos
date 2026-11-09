# Covariance Factorisation

Also called: Covariance Decomposition, RQ Decomposition for Covariance, Rotation and Scale Parameterization

## Overview
Covariance Factorisation is a parameterization technique for 3D Gaussian splats that decomposes the covariance matrix into a rotation matrix and a scaling matrix, ensuring the covariance remains positive semi-definite under unconstrained gradient descent [S1, unofficial]. It sets itself apart from direct covariance optimization by embedding geometric constraints directly into the parameter space, and from Cholesky parameterization by explicitly separating rotation and scale for physical interpretability [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
- Input: Unconstrained learnable parameters for rotation (typically a quaternion or axis-angle) and scale (3D vector).
- Output: The 3D covariance matrix $\boldsymbol{\Sigma}$.

**Data Structures:**
- Rotation $\mathbf{R}$: A $3 \times 3$ orthonormal matrix derived from a 4D quaternion $\mathbf{q} = [w, x, y, z]$ or similar representation.
- Scale $\mathbf{S}$: A $3 \times 3$ diagonal matrix $\text{diag}(s_x, s_y, s_z)$ where $s_i > 0$.

**Steps:**
1. **Parameterize Rotation:** Convert the learnable rotation parameters (e.g., quaternion) into a $3 \times 3$ rotation matrix $\mathbf{R}$. Ensure $\mathbf{R}$ is orthonormal ($\mathbf{R}^T \mathbf{R} = \mathbf{I}$).
2. **Parameterize Scale:** Convert the learnable scale parameters into a diagonal scaling matrix $\mathbf{S} = \text{diag}(s_x, s_y, s_z)$. To ensure positive scale, apply an exponential or absolute value function if the raw parameters are unconstrained [G].
3. **Compute Covariance:** Calculate the covariance matrix using the formula:
   $$ \boldsymbol{\Sigma} = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T $$
   Since $\mathbf{S}$ is diagonal, $\mathbf{S} \mathbf{S}^T = \text{diag}(s_x^2, s_y^2, s_z^2)$. Thus:
   $$ \boldsymbol{\Sigma} = \mathbf{R} \begin{bmatrix} s_x^2 & 0 & 0 \\ 0 & s_y^2 & 0 \\ 0 & 0 & s_z^2 \end{bmatrix} \mathbf{R}^T $$
4. **Optimization:** During gradient descent, optimize the unconstrained rotation and scale parameters. The factorization guarantees that $\boldsymbol{\Sigma}$ remains positive semi-definite by construction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_covariance(quaternion: Tuple[float, float, float, float], scales: Tuple[float, float, float]) -> Matrix3x3:
    """
    Computes the 3D covariance matrix from quaternion and scale parameters.
    
    Args:
        quaternion: (w, x, y, z) normalized quaternion representing rotation.
        scales: (sx, sy, sz) positive scale factors.
        
    Returns:
        3x3 covariance matrix Sigma.
    """
    # 1. Convert quaternion to rotation matrix R
    w, x, y, z = quaternion
    # Ensure normalization
    norm = sqrt(w*w + x*x + y*y + z*z)
    w, x, y, z = w/norm, x/norm, y/norm, z/norm
    
    # Standard quaternion to rotation matrix conversion
    R = Matrix3x3(
        1 - 2*(y*y + z*z), 2*(x*y - w*z),     2*(x*z + w*y),
        2*(x*y + w*z),     1 - 2*(x*x + z*z), 2*(y*z - w*x),
        2*(x*z - w*y),     2*(y*z + w*x),     1 - 2*(x*x + y*y)
    )
    
    # 2. Construct scaling matrix S (diagonal)
    sx, sy, sz = scales
    # Ensure positive scales for physical significance [G]
    sx = exp(sx)
    sy = exp(sy)
    sz = exp(sz)
    
    # 3. Compute Sigma = R * S * S^T * R^T
    # S * S^T is diagonal with squared scales
    S_sq = Matrix3x3(
        sx*sx, 0, 0,
        0, sy*sy, 0,
        0, 0, sz*sz
    )
    
    # Matrix multiplication: R @ S_sq @ R^T
    # First compute R @ S_sq
    R_S_sq = Matrix3x3(
        R[0][0]*sx*sx, R[0][1]*sy*sy, R[0][2]*sz*sz,
        R[1][0]*sx*sx, R[1][1]*sy*sy, R[1][2]*sz*sz,
        R[2][0]*sx*sx, R[2][1]*sy*sy, R[2][2]*sz*sz
    )
    
    # Then (R @ S_sq) @ R^T
    Sigma = Matrix3x3(
        R_S_sq[0][0]*R[0][0] + R_S_sq[0][1]*R[1][0] + R_S_sq[0][2]*R[2][0],
        R_S_sq[0][0]*R[0][1] + R_S_sq[0][1]*R[1][1] + R_S_sq[0][2]*R[2][1],
        R_S_sq[0][0]*R[0][2] + R_S_sq[0][1]*R[1][2] + R_S_sq[0][2]*R[2][2],
        
        R_S_sq[1][0]*R[0][0] + R_S_sq[1][1]*R[1][0] + R_S_sq[1][2]*R[2][0],
        R_S_sq[1][0]*R[0][1] + R_S_sq[1][1]*R[1][1] + R_S_sq[1][2]*R[2][1],
        R_S_sq[1][0]*R[0][2] + R_S_sq[1][1]*R[1][2] + R_S_sq[1][2]*R[2][2],
        
        R_S_sq[2][0]*R[0][0] + R_S_sq[2][1]*R[1][0] + R_S_sq[2][2]*R[2][0],
        R_S_sq[2][0]*R[0][1] + R_S_sq[2][1]*R[1][1] + R_S_sq[2][2]*R[2][1],
        R_S_sq[2][0]*R[0][2] + R_S_sq[2][1]*R[1][2] + R_S_sq[2][2]*R[2][2]
    )
    
    return Sigma
```

**Edge Cases and Pitfalls:**
- **Degenerate Scales:** If any scale parameter approaches zero, the Gaussian becomes degenerate (flat). Exponential parameterization prevents negative scales but can lead to numerical underflow if scales are too small [G].
- **Quaternion Ambiguity:** Quaternions $\mathbf{q}$ and $-\mathbf{q}$ represent the same rotation. Optimization may jump between these, causing discontinuities. Use a consistent convention or local parameterization [G].
- **Orthonormality Drift:** If $\mathbf{R}$ is not strictly orthonormal, $\boldsymbol{\Sigma}$ may not be positive semi-definite. Re-orthonormalization (e.g., via SVD) may be needed after updates [G].

## Subtopics

### [Cholesky Parameterisation for Gaussian Splats](3120-cholesky-parameterisation-for-gaussian-splats.md)
Cholesky Parameterisation represents the covariance matrix using the lower triangular Cholesky factor, guaranteeing positive-definiteness by construction. It sets itself apart from quaternion-based factorizations by storing fewer parameters ($D(D+1)/2$) and from unconstrained optimization by embedding geometric constraints directly into the parameter space. It is primarily used in Luxar for microscopy data visualization to ensure stable optimization of Gaussian splats.

### Gamma-Decomposed Covariance Update
A numerical stability technique for computing the new covariance matrix after recycling, replacing the unsupported $\beta()$ function with $\Gamma()$ and $\ln(\Gamma())$ to prevent overflow [S3].

## Operating characteristics
- **Time Complexity:** $O(1)$ per Gaussian for covariance computation (constant size matrix operations).
- **Memory Complexity:** $O(1)$ per Gaussian for storing rotation (4 floats) and scale (3 floats) instead of 6 floats for the full covariance matrix.
- **Strengths:** Ensures positive semi-definiteness without projection; physically interpretable rotation and scale; efficient gradient computation.
- **Limitations:** Requires careful handling of rotation parameterization to avoid singularities; scale parameterization must ensure positivity.
- **When to Use:** Standard 3D Gaussian Splatting optimization where physical interpretability and stability are required.
- **When Not to Use:** When memory is extremely constrained and Cholesky parameterization is preferred for fewer parameters.

## Background
Originated in the 3D Gaussian Splatting paper [S1, unofficial] as a core component of the rendering pipeline. It enables efficient optimization of anisotropic 3D Gaussians by decoupling rotation and scale. The method has been adopted in various extensions and implementations of 3DGS [S2].

## Sources
[S1] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
