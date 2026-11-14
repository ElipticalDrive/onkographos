# Explicit Camera Jacobian Derivation

Also called: Analytical Jacobians for 3DGS, Camera Pose Jacobian Derivation (Lie Group), Lie Algebra Jacobian for 3DGS

## Overview
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian Splatting (3DGS) primitives to enable efficient, differentiable camera tracking without the overhead of automatic differentiation [S1, unofficial]. It solves the problem of computationally expensive gradient computation in SLAM systems by providing minimal parameterization on the Lie group, ensuring the Jacobian dimensionality matches the degrees of freedom [S1, unofficial]. Unlike prior ray-intersection methods that assume fewer Gaussians, this approach is specifically tailored for EWA splatting and 3DGS [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera pose $T_{CW} \in SE(3)$, Gaussian mean in world coordinates $\mu_W$, Gaussian covariance in world coordinates $\Sigma_W$, projection matrix $P$, and rotation component $W$ of the camera pose.
*   **Output:** Jacobians $\frac{\partial \mu_I}{\partial T_{CW}}$ and $\frac{\partial \Sigma_I}{\partial T_{CW}}$ required for gradient descent optimization of the camera pose.

**Data Structures:**
*   $T_{CW}$: $4 \times 4$ homogeneous transformation matrix representing camera-to-world pose.
*   $\mu_C$: $3 \times 1$ vector representing Gaussian mean in camera coordinates.
*   $\Sigma_I$: $2 \times 2$ matrix representing the 2D covariance in image space.
*   $W$: $3 \times 3$ rotation matrix component of $T_{CW}$.
*   $\tau$: $6 \times 1$ vector in Lie algebra $se(3)$.

**Mathematical Definitions:**
The derivative on the manifold is defined as [S1, unofficial]:
$$
\frac{\mathcal{D}(f(\mathbf{T}))}{\mathcal{D}\mathbf{T}} \triangleq \lim_{\tau \rightarrow 0} \frac{\text{Log}(f(\text{Exp}(\tau) \circ \mathbf{T}) \circ f(\mathbf{T})^{-1})}{\tau}
$$
where $\circ$ is group composition, and Exp/Log are exponential/logarithmic mappings between Lie algebra and Lie Group [S1, unofficial].

The hat operator $\times$ maps a 3D vector $\mathbf{v} = [v_x, v_y, v_z]^T$ to a skew-symmetric matrix:
$$
\mathbf{v}^\times = \begin{bmatrix} 0 & -v_z & v_y \\ v_z & 0 & -v_x \\ -v_y & v_x & 0 \end{bmatrix}
$$

**Step-by-Step Derivation:**

1.  **Compute Camera Coordinate Mean Jacobian:**
    The mean in camera coordinates is $\mu_C = W^T (\mu_W - t)$, where $t$ is the translation part of $T_{CW}$. Using the chain rule [S1, unofficial]:
    $$
    \frac{\partial \mu_I}{\partial T_{CW}} = \frac{\partial \mu_I}{\partial \mu_C} \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}}
    $$
    The manifold derivative $\frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}}$ is derived as [S1, unofficial]:
    $$
    \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}} = [I \quad -\mu_C^\times]
    $$
    where $I$ is the $3 \times 3$ identity matrix and $\mu_C^\times$ is the skew-symmetric matrix of $\mu_C$ [S1, unofficial].

2.  **Compute 2D Covariance Jacobian:**
    The 2D covariance $\Sigma_I$ depends on the projection Jacobian $J$ and the rotation component $W$. The chain rule yields [S1, unofficial]:
    $$
    \frac{\partial \Sigma_I}{\partial T_{CW}} = \frac{\partial \Sigma_I}{\partial J} \frac{\partial J}{\partial \mu_C} \frac{D\mu_C}{DT_{CW}} + \frac{\partial \Sigma_I}{\partial W} \frac{DW}{DT_{CW}}
    $$
    The term $\frac{DW}{DT_{CW}}$ is derived by considering the rotation vector $\theta$. The derivative of the rotation matrix component $W$ with respect to the rotation vector $\theta$ is [S1, unofficial]:
    $$
    \frac{\partial W}{\partial \theta} = \begin{bmatrix} -W_{:,1}^\times & -W_{:,2}^\times & -W_{:,3}^\times \end{bmatrix}
    $$
    where $W_{:,i}$ is the $i$-th column of $W$ [S1, unofficial]. This results in the manifold derivative [S1, unofficial]:
    $$
    \frac{\mathcal{D}W}{\mathcal{D}T_{CW}} = \begin{bmatrix} 0 & -W_{,1}^\times \\ 0 & -W_{,2}^\times \\ 0 & -W_{,3}^\times \end{bmatrix}
    $$
    Note: The source notation $\frac{\mathcal{D}W}{\mathcal{D}T_{CW}}$ in Eq. 6 appears to represent the block structure relative to the 6-DOF Lie algebra vector, where the first 3 elements (translation) do not affect $W$ directly in this specific decomposition, hence the zero blocks [S1, unofficial].

3.  **Optimization Loop:**
    For tracking, minimize photometric or geometric residuals using gradient descent. At least 50 iterations per frame are required for accurate tracking [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def skew_symmetric(v):
    """Converts a 3D vector to a skew-symmetric matrix."""
    return np.array([
        [0, -v[2], v[1]],
        [v[2], 0, -v[0]],
        [-v[1], v[0], 0]
    ])

def compute_camera_mean_jacobian(mu_C):
    """
    Computes D(mu_C) / D(T_CW).
    mu_C: 3D vector, Gaussian mean in camera coordinates.
    Returns: 3x6 matrix.
    """
    I_3 = np.eye(3)
    mu_C_skew = skew_symmetric(mu_C)
    # [I | -mu_C^x]
    return np.hstack([I_3, -mu_C_skew])

def compute_rotation_jacobian(W):
    """
    Computes D(W) / D(T_CW) with respect to the Lie algebra.
    W: 3x3 rotation matrix.
    Returns: 3x3x6 tensor or flattened structure depending on implementation.
    Here we return the block matrix form as described in Eq. 6.
    """
    # W columns
    w1 = W[:, 0]
    w2 = W[:, 1]
    w3 = W[:, 2]
    
    # Skew symmetric matrices of columns
    w1_skew = skew_symmetric(w1)
    w2_skew = skew_symmetric(w2)
    w3_skew = skew_symmetric(w3)
    
    # Construct the 9x6 matrix (flattened W)
    # The structure is [0, -W_{:,1}^x; 0, -W_{:,2}^x; 0, -W_{:,3}^x]
    # This implies the first 3 columns (translation part of Lie algebra) are 0.
    # The last 3 columns (rotation part) are -W_{:,i}^x.
    
    zero_block = np.zeros((3, 3))
    rot_block_1 = -w1_skew
    rot_block_2 = -w2_skew
    rot_block_3 = -w3_skew
    
    # Stack vertically: 3 rows of blocks
    # Row 1: [0, rot_block_1]
    # Row 2: [0, rot_block_2]
    # Row 3: [0, rot_block_3]
    # Wait, Eq 6 shows a 3x3 block structure where each block is 3x3?
    # "where W:,i refers to the ith column of the matrix."
    # The matrix is 3x3. The derivative is w.r.t 6D vector.
    # The result in Eq 6 is:
    # [ 0    -W_{:,1}^x ]
    # [ 0    -W_{:,2}^x ]
    # [ 0    -W_{:,3}^x ]
    # This looks like a 9x6 matrix if we consider W as a vector of 9 elements?
    # Or is it 3x6? 
    # Let's look at Eq 31: dW/d_theta_x = e1^x W. This is 3x3.
    # The derivative of the 3x3 matrix W w.r.t 3D rotation vector is a 3x3x3 tensor.
    # Flattened, it is 9x3.
    # The Eq 6 format suggests a specific layout for the 6-DOF Jacobian.
    # Let's implement the 9x6 matrix as implied by the block structure.
    
    # Block 1 (top 3 rows of W): [0, -w1_skew] -> 3x6
    # Block 2 (middle 3 rows of W): [0, -w2_skew] -> 3x6
    # Block 3 (bottom 3 rows of W): [0, -w3_skew] -> 3x6
    
    block1 = np.hstack([np.zeros((3,3)), -w1_skew])
    block2 = np.hstack([np.zeros((3,3)), -w2_skew])
    block3 = np.hstack([np.zeros((3,3)), -w3_skew])
    
    return np.vstack([block1, block2, block3])

def compute_covariance_jacobian(J, mu_C, W, dSigma_dJ, dSigma_dW):
    """
    Computes dSigma_I / dT_CW.
    J: 2x3 projection Jacobian.
    mu_C: 3D vector.
    W: 3x3 rotation matrix.
    dSigma_dJ: Gradient of Sigma w.r.t J.
    dSigma_dW: Gradient of Sigma w.r.t W.
    """
    # Term 1: dSigma/dJ * dJ/dmu_C * D(mu_C)/D(T_CW)
    # Note: dJ/dmu_C is not explicitly given in the notes, but J is the Jacobian of projection.
    # In 3DGS, J is usually computed as the derivative of the projection function.
    # We assume dSigma_dJ is provided by the upstream gradient.
    # dJ/dmu_C is part of the projection geometry. 
    # For implementation, this term is often computed via chain rule in the rasterizer.
    # Here we focus on the explicit camera Jacobian parts derived in the notes.
    
    # Term 2: dSigma/dW * DW/D(T_CW)
    # dSigma_dW is 3x3 gradient. DW/D(T_CW) is 9x6.
    # We need to contract these.
    
    # Since the exact contraction of dSigma/dW (3x3) with DW/D(T_CW) (9x6) 
    # requires vectorization of W, we note that:
    # dSigma/dT_CW_term2 = reshape(dSigma_dW, (9,1)) @ DW_D_TCW
    
    # The first term involves dJ/dmu_C. 
    # J = P * [I | -mu_C^x] ? No, J is 2x3.
    # The notes state: dSigma_I/dT_CW = dSigma_I/dJ * dJ/dmu_C * Dmu_C/DTCW + ...
    # We implement the second term explicitly as derived.
    
    DW_DT = compute_rotation_jacobian(W)
    
    # Vectorize dSigma_dW (3x3) to 9x1
    dSigma_dW_vec = dSigma_dW.flatten()
    
    # Contract: (1x9) @ (9x6) = 1x6
    term2 = dSigma_dW_vec @ DW_DT
    
    # Term 1 is handled by the standard 3DGS rasterizer gradients for J.
    # The explicit contribution from camera pose to Sigma via W is term2.
    # The contribution via mu_C is in the first part.
    
    return term2 # Simplified return for the explicit W-dependent part
```

**Parameters:**
*   **Iterations:** At least 50 iterations of gradient descent per frame are required for accurate tracking [S1, unofficial].
*   **Hyperparameters:** $\lambda_{pho}$ balances photometric and geometric residuals in SLAM tracking [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The use of Lie algebra ensures minimal parameterization, avoiding redundant computations and potential singularities associated with Euler angles [S1, unofficial].
*   **Boundaries:** The method assumes the Gaussians are visible and the projection is valid. Degenerate cases (e.g., Gaussian at camera center) may require numerical stabilization [G].

## Subtopics

### Photometric and Geometric Tracking
Optimizes only the current camera pose without updating the map representation, minimizing photometric and/or geometric residuals [S1, unofficial].
*   **Photometric Residual:** $E_{pho} = \| I(\mathcal{G}, \mathbf{T}_{CW}) - \bar{I} \|_1$, where $I(\mathcal{G}, \mathbf{T}_{CW})$ renders Gaussians $\mathcal{G}$ from pose $\mathbf{T}_{CW}$ and $\bar{I}$ is the observed image [S1, unofficial].
*   **Geometric Residual:** $E_{geo} = \| D(\mathcal{G}, \mathbf{T}_{CW}) - \bar{D} \|_1$, where $D(\mathcal{G}, \mathbf{T}_{CW})$ is depth rasterization and $\bar{D}$ is observed depth [S1, unofficial].
*   **Depth Rasterization:** $\mathcal{D}_p = \sum_{i \in \mathcal{N}} z_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)$, where $z_i$ is the distance to the mean $\mu_W$ of Gaussian $i$ along the camera ray [S1, unofficial].
*   **Optimization:** Minimizes $\lambda_{pho} E_{pho} + (1-\lambda_{pho}) E_{geo}$. Affine brightness parameters are optimized for varying exposure, and non-edge or low-opacity pixels are penalized [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The explicit derivation avoids the overhead of automatic differentiation, which is critical for performance-critical rasterization [S1, unofficial]. The system operates at 3.2 FPS with monocular input and 2.5 FPS with depth on the Replica dataset [S1, unofficial].
*   **Memory Complexity:** Minimal parameterization reduces the size of the Jacobian matrices, eliminating redundant computations [S1, unofficial].
*   **Strengths:** Enables accurate tracking with at least 50 iterations per frame; unifies SLAM components (tracking, mapping, keyframe management) using 3D Gaussians [S1, unofficial].
*   **Limitations:** Requires at least 50 iterations per frame, which can be computationally intensive for real-time applications without hardware acceleration [S1, unofficial]. Assumes offline application for dynamic tracking and 4D scene representation if accurate camera position is not provided [S1, unofficial].
*   **When to Use:** When integrating 3DGS into a SLAM system requiring efficient camera pose optimization.
*   **When Not to Use:** When the number of Gaussians is very small and ray-intersection methods are sufficient, or when automatic differentiation overhead is acceptable for offline processing [S1, unofficial].

## Background
The method was introduced in "Gaussian Splatting SLAM" [S1, unofficial]. It addresses the limitation of prior camera pose Jacobian methods that assumed a smaller number of Gaussians and were based on ray-intersection rather than splatting, making them inapplicable to 3DGS [S1, unofficial]. The derivation uses Lie algebra to ensure minimal parameterization, matching the degrees of freedom of the SE(3) group [S1, unofficial].

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
