# Spatial-Orthogonal Cholesky Parameterization

Also called: Hybrid Cholesky-Skew-Symmetric parameterization, UBS covariance parameterization

## Overview
Spatial-Orthogonal Cholesky Parameterization is a method for parameterizing the covariance matrix of N-dimensional Gaussian splats that combines Cholesky factorization with skew-symmetric rotation matrices to ensure positive semi-definiteness while preserving explicit 3D spatial interpretability [S1, unofficial]. It solves the limitation of pure Cholesky parameterization, which loses explicit spatial rotation structure, and the limitation of quaternion parameterization, which does not generalize beyond 3D [S1, unofficial]. It is used in Universal Beta Splatting (UBS) to enable flexible cross-dimensional correlations with non-spatial dimensions (e.g., view, time) while maintaining efficient optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Learnable parameters for a single splat:
  - $\boldsymbol{\omega} \in \mathbb{R}^3$: Skew-symmetric rotation vector.
  - $\mathbf{s}_x \in \mathbb{R}^3$: Spatial scale vector.
  - $\mathbf{s}_q \in \mathbb{R}^C$: Non-spatial scale vector (for $C$ additional dimensions).
  - $\mathbf{L}_{qx} \in \mathbb{R}^{C \times 3}$: Cross-dimensional correlation matrix.
  - $\mathbf{L}_q \in \mathbb{R}^{C \times C}$: Lower-triangular matrix for non-spatial dimensions (Cholesky factor).
- **Output:** The $N \times N$ covariance matrix $\Sigma$, where $N = 3 + C$.

**Data Structures:**
The covariance matrix $\Sigma$ is constructed as $\Sigma = L L^\top$, where $L$ is a structured lower-triangular matrix of size $N \times N$:
$$
L = \begin{pmatrix} R_x \text{diag}(\mathbf{s}_x) & \mathbf{0}_{3 \times C} \\ \mathbf{L}_{qx} & \mathbf{L}_q \end{pmatrix}
$$
Here, $R_x \in SO(3)$ is the spatial rotation matrix, $\text{diag}(\mathbf{s}_x)$ is a $3 \times 3$ diagonal matrix of spatial scales, $\mathbf{0}_{3 \times C}$ is a zero matrix, $\mathbf{L}_{qx}$ encodes correlations between spatial and non-spatial dimensions, and $\mathbf{L}_q$ is a lower-triangular matrix for the non-spatial subspace.

**Steps:**
1. **Construct Skew-Symmetric Matrix:** Create $A_x \in \mathbb{R}^{3 \times 3}$ from $\boldsymbol{\omega} = (\omega_1, \omega_2, \omega_3)^\top$:
   $$
   A_x = \begin{pmatrix} 0 & -\omega_3 & \omega_2 \\ \omega_3 & 0 & -\omega_1 \\ -\omega_2 & \omega_1 & 0 \end{pmatrix}
   $$
2. **Compute Rotation Matrix:** Approximate $R_x$ using the first-order Taylor expansion of the matrix exponential:
   $$
   R_x \approx I + A_x
   $$
   This approximation maintains approximate orthogonality for small $\|\boldsymbol{\omega}\|$ and initializes with identity rotation ($\boldsymbol{\omega} = \mathbf{0}$) [S1, unofficial].
3. **Construct Spatial Block:** Compute the spatial part of $L$:
   $$
   L_{xx} = R_x \text{diag}(\mathbf{s}_x)
   $$
4. **Assemble L:** Combine blocks into the full lower-triangular matrix $L$:
   $$
   L = \begin{pmatrix} L_{xx} & \mathbf{0} \\ \mathbf{L}_{qx} & \mathbf{L}_q \end{pmatrix}
   $$
5. **Compute Covariance:** Calculate $\Sigma = L L^\top$. The expanded form is:
   $$
   \Sigma = \begin{pmatrix} R_x \text{diag}(\mathbf{s}_x)^2 R_x^\top & R_x \text{diag}(\mathbf{s}_x) \mathbf{L}_{qx}^\top \\ \mathbf{L}_{qx} \text{diag}(\mathbf{s}_x) R_x^\top & \mathbf{L}_{qx} \mathbf{L}_{qx}^\top + \mathbf{L}_q \mathbf{L}_q^\top \end{pmatrix}
   $$

**Parameters:**
- $\boldsymbol{\omega}$: Initialized to $\mathbf{0}$.
- $\mathbf{s}_x, \mathbf{s}_q$: Initialized to positive values (e.g., small constants or derived from initial splat size).
- $\mathbf{L}_{qx}$: Initialized to $\mathbf{0}$.
- $\mathbf{L}_q$: Initialized as a lower-triangular matrix with positive diagonal entries.

**Edge Cases and Pitfalls:**
- The first-order Taylor approximation $R_x \approx I + A_x$ is only valid for small $\|\boldsymbol{\omega}\|$. Large rotations may lead to non-orthogonal $R_x$, potentially affecting numerical stability [S1, unofficial].
- $\mathbf{L}_q$ must remain lower-triangular with positive diagonal entries to ensure $\Sigma$ is positive semi-definite.

## Operating characteristics
- **Time Complexity:** Efficient due to the first-order Taylor approximation for rotation, avoiding expensive matrix exponential computations [S1, unofficial].
- **Memory Complexity:** Stores $3 + 3 + C + 3C + C(C+1)/2$ parameters per splat. For $C=0$ (pure 3D), this is 6 parameters, same as standard 3DGS. For $C>0$, it scales with $C^2$.
- **Performance:** On Mip-NeRF 360 (UBS-3D), achieves 25.54 dB PSNR with 1 hour training time, outperforming Quaternion (25.50 dB, 3h26m), Cholesky (25.28 dB, 1h14m), and Cayley Transform (25.52 dB, 1h16m) [S1, unofficial].
- **Strengths:** Generalizes to arbitrary N dimensions, preserves 3D spatial interpretability, enables cross-dimensional correlations, and is computationally efficient [S1, unofficial].
- **Limitations:** The first-order Taylor approximation may not capture large rotations accurately. Pure Cholesky loses spatial structure, and Quaternions do not generalize beyond 3D [S1, unofficial].
- **When to Use:** When rendering N-dimensional radiance fields (e.g., with view or time dimensions) and requiring explicit 3D spatial structure and cross-dimensional correlations.
- **When Not to Use:** When only 3D spatial rendering is needed and simplicity is preferred (standard 3DGS or Cholesky may suffice), or when large rotations are expected and higher-order rotation parameterizations are needed.

## Background
Originated in Universal Beta Splatting (UBS) [S1, unofficial]. It addresses the challenge of parameterizing covariance matrices in N-dimensional radiance field rendering, combining the benefits of Cholesky factorization (PSD guarantee, generalization) and skew-symmetric rotation (spatial interpretability) [S1, unofficial].

## Sources
[S1] Universal Beta Splatting, https://doi.org/10.48550/arXiv.2510.03312, fetched 2026-10-30 (unofficial)
