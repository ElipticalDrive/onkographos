# Explicit Camera Jacobian Derivation

Also called: Analytical Jacobians for 3DGS, Camera Pose Jacobian Derivation (Lie Group), Lie Algebra Jacobian for 3DGS

## Overview
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian primitives to enable efficient camera pose optimization in SLAM systems. It solves the problem of high computational overhead associated with automatic differentiation in differentiable rasterization, specifically for the EWA splatting and 3D Gaussian Splatting (3DGS) rendering pipeline. Unlike prior ray-intersection methods that assume fewer Gaussians, this approach provides the first analytical Jacobian for 3DGS, ensuring minimal parameterization by matching the Jacobian dimensionality to the degrees of freedom of the Lie algebra [S1, unofficial].

## How it works
The method computes the gradient of the rendered image or depth with respect to the camera pose $T_{CW} \in SE(3)$ by applying the chain rule through the Gaussian mean $\mu_C$ and covariance $\Sigma_I$ in camera coordinates. It uses Lie algebra to derive minimal Jacobians, eliminating redundant computations inherent in non-minimal parameterizations [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Camera pose $T_{CW} \in SE(3)$, 3D Gaussian parameters in world coordinates (mean $\mu_W$, covariance $\Sigma_W$), projection matrix $P$.
*   **Output:** Jacobian matrices $\frac{\partial \mu_I}{\partial T_{CW}}$ and $\frac{\partial \Sigma_I}{\partial T_{CW}}$ required for gradient descent optimization of the camera pose.

**Data Structures:**
*   $T_{CW}$: $4 \times 4$ homogeneous transformation matrix representing camera-to-world pose.
*   $\mu_C$: $3 \times 1$ vector representing Gaussian mean in camera coordinates.
*   $W$: $3 \times 3$ rotation matrix component of $T_{CW}$.
*   $\tau$: $6 \times 1$ vector in Lie algebra $se(3)$, representing infinitesimal perturbation.
*   $\theta$: $3 \times 1$ rotation vector component of $\tau$.
*   $\rho$: $3 \times 1$ translation vector component of $\tau$.

**Mathematical Definitions:**
The derivative on the manifold is defined as [S1, unofficial]:
$$
\frac{\mathcal{D}(f(\mathbf{T}))}{\mathcal{D}\mathbf{T}} \triangleq \lim_{\tau \rightarrow 0} \frac{\text{Log}(f(\text{Exp}(\tau) \circ \mathbf{T}) \circ f(\mathbf{T})^{-1})}{\tau}
$$
where $\circ$ is group composition, and $\text{Exp}, \text{Log}$ are the exponential and logarithmic mappings between Lie algebra $se(3)$ and Lie group $SE(3)$.

The hat operator $\times$ maps a 3D vector $\mathbf{v} = [v_x, v_y, v_z]^T$ to a skew-symmetric matrix:
$$
\mathbf{v}^\times = \begin{bmatrix} 0 & -v_z & v_y \\ v_z & 0 & -v_x \\ -v_y & v_x & 0 \end{bmatrix}
$$

**Step-by-Step Derivation:**

1.  **Chain Rule Application:**
    The total derivative with respect to camera pose is decomposed using the chain rule [S1, unofficial]:
    $$
    \frac{\partial \mu_I}{\partial T_{CW}} = \frac{\partial \mu_I}{\partial \mu_C} \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}}
    $$
    $$
    \frac{\partial \Sigma_I}{\partial T_{CW}} = \frac{\partial \Sigma_I}{\partial J} \frac{\partial J}{\partial \mu_C} \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}} + \frac{\partial \Sigma_I}{\partial W} \frac{\mathcal{D}W}{\mathcal{D}T_{CW}}
    $$
    where $\mu_I$ is the projected mean, $\Sigma_I$ is the 2D covariance, and $J$ is the Jacobian of the projection operation.

2.  **Derivative of Camera Coordinate Mean:**
    The mean in camera coordinates is $\mu_C = W \mu_W + t$, where $t$ is the translation part of $T_{CW}$. The manifold derivative is derived as [S1, unofficial]:
    $$
    \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}} = \lim_{\tau \rightarrow 0} \frac{\text{Exp}(\tau) \cdot \mu_C - \mu_C}{\tau}
    $$
    Expanding $\text{Exp}(\tau) \approx I + \tau^\wedge$ for small $\tau$:
    $$
    = \lim_{\tau \rightarrow 0} \frac{(I + \tau^\wedge) \mu_C - \mu_C}{\tau} = \lim_{\tau \rightarrow 0} \frac{\tau^\wedge \mu_C}{\tau}
    $$
    Decomposing $\tau^\wedge \mu_C = \theta^\times \mu_W + \rho$ (where $\rho$ is the translational part of $\tau$ acting on the origin, but here acting on the vector relative to the rotation center, effectively $\rho$ is the translation perturbation):
    $$
    = \lim_{\tau \rightarrow 0} \frac{-\mu_C^\times \theta + \rho}{\tau}
    $$
    This yields the minimal Jacobian [S1, unofficial]:
    $$
    \frac{\mathcal{D}\mu_C}{\mathcal{D}T_{CW}} = \begin{bmatrix} I & -\mu_C^\times \end{bmatrix}
    $$
    where $I$ is the $3 \times 3$ identity matrix and $-\mu_C^\times$ is the skew-symmetric matrix of the negative camera-space mean.

3.  **Derivative of Rotation Component:**
    The rotation matrix $W$ depends on the rotation vector $\theta$. The derivative of the rotation matrix component $W$ with respect to the camera pose is derived using the property $\frac{\partial W}{\partial \theta_i} = \mathbf{e}_i^\times W$, where $\mathbf{e}_i$ are basis vectors [S1, unofficial]:
    $$
    \frac{\mathcal{D}W}{\mathcal{D}T_{CW}} = \begin{bmatrix} 0 & -W_{:,1}^\times \\ 0 & -W_{:,2}^\times \\ 0 & -W_{:,3}^\times \end{bmatrix}
    $$
    Here, $W_{:,i}$ is the $i$-th column of the rotation matrix $W$. The top row is zero because the translation part of the pose does not directly affect the rotation matrix structure in this minimal parameterization context, or rather, the derivative is structured to map the 6D Lie algebra vector to the change in $W$. Specifically, the derivative of $W$ with respect to $\theta$ is a $3 \times 3 \times 3$ tensor, but flattened for the Jacobian structure relative to the 6D pose vector. The formula provided in the source represents the block structure where the first 3 columns (translation) are zero, and the last 3 columns (rotation) contain the skew-symmetric products of the columns of $W$ [S1, unofficial].

    *Correction/Clarification based on source Eq 6:* The source presents $\frac{\mathcal{D}W}{\mathcal{D}T_{CW}}$ as a matrix. Given $W$ is $3 \times 3$, and $T_{CW}$ is 6-DOF, this derivative is likely part of the chain rule for $\Sigma_I$. The explicit form given is:
    $$
    \frac{\mathcal{D}W}{\mathcal{D}T_{CW}} = \begin{bmatrix} 0 & -W_{,1}^\times \\ 0 & -W_{,2}^\times \\ 0 & -W_{,3}^\times \end{bmatrix}
    $$
    This notation implies a $9 \times 6$ matrix if $W$ is vectorized, or it represents the coefficients for the change in $W$. The source states $W_{:,i}$ is the $i$-th column. The term $-W_{:,i}^\times$ is a $3 \times 3$ skew-symmetric matrix. The structure suggests that for each column of $W$, the derivative with respect to the rotation part of the pose involves the skew-symmetric product.

4.  **Jacobian of Rotation Matrix w.r.t Rotation Vector:**
    The derivative of the rotation matrix $\mathbf{W}$ with respect to the rotation vector $\theta$ components is:
    $$
    \frac{\partial \mathbf{W}}{\partial \theta_x} = \mathbf{e}_1^\times \mathbf{W} = \begin{bmatrix} \mathbf{0}_{1 \times 3} \\ -\mathbf{W}_{3,:} \\ \mathbf{W}_{2,:} \end{bmatrix}
    $$
    $$
    \frac{\partial \mathbf{W}}{\partial \theta_y} = \mathbf{e}_2^\times \mathbf{W}
    $$
    $$
    \frac{\partial \mathbf{W}}{\partial \theta_z} = \mathbf{e}_3^\times \mathbf{W}
    $$
    where $\mathbf{e}_1 = [1, 0, 0]^T$, etc. [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def skew_symmetric(v):
    """
    Computes the skew-symmetric matrix for a 3D vector v.
    v: array-like, shape (3,)
    Returns: ndarray, shape (3, 3)
    """
    return np.array([
        [0, -v[2], v[1]],
        [v[2], 0, -v[0]],
        [-v[1], v[0], 0]
    ])

def compute_camera_jacobians(mu_C, W):
    """
    Computes the minimal analytical Jacobians for camera pose optimization.
    
    mu_C: ndarray, shape (3,), Gaussian mean in camera coordinates.
    W: ndarray, shape (3, 3), Rotation matrix component of camera pose.
    
    Returns:
    d_mu_C_d_T: ndarray, shape (3, 6), Jacobian of camera-space mean w.r.t. SE(3) pose.
    d_W_d_T: ndarray, shape (9, 6), Jacobian of rotation matrix w.r.t. SE(3) pose.
               Note: The source formula for d_W/d_T is presented in a specific block form.
               We reconstruct the 9x6 matrix based on the source's Eq 6 and derivation.
    """
    # 1. Jacobian of mu_C w.r.t. T_CW
    # Formula: [I, -mu_C^x]
    I_3 = np.eye(3)
    mu_C_skew = skew_symmetric(mu_C)
    d_mu_C_d_T = np.hstack([I_3, -mu_C_skew]) # Shape (3, 6)
    
    # 2. Jacobian of W w.r.t. T_CW
    # Formula from source: 
    # [ 0, -W_{:,1}^x ]
    # [ 0, -W_{:,2}^x ]
    # [ 0, -W_{:,3}^x ]
    # This structure is ambiguous in standard matrix notation. 
    # Based on Eq 31 and the limit derivation, the derivative of W (3x3) w.r.t theta (3x1) 
    # is a 9x3 matrix. The derivative w.r.t translation is 0.
    # So d_W_d_T is (9, 6).
    # The first 3 columns (translation) are zero.
    # The last 3 columns (rotation) are derived from dW/d_theta.
    
    # dW/d_theta_i = e_i^x W
    # We need to stack these derivatives for i=1,2,3 into a 9x3 matrix.
    # Let J_W_theta be the 9x3 matrix where each block of 3 rows corresponds to dW/d_theta_i.
    
    # Extract columns of W
    W_col1 = W[:, 0]
    W_col2 = W[:, 1]
    W_col3 = W[:, 2]
    
    # Compute skew-symmetric matrices for the columns
    # Note: The source Eq 31 shows dW/d_theta_x = e1^x W.
    # e1^x = [[0,0,0],[0,0,-1],[0,1,0]]
    # e1^x W = [[0,0,0], [-W[2,0], -W[2,1], -W[2,2]], [W[1,0], W[1,1], W[1,2]]]
    # This results in a 3x3 matrix. We vectorize it to 9x1.
    
    def dW_d_theta_i(i):
        """Returns the 9x1 vectorized derivative of W w.r.t theta_i."""
        e_i = np.zeros(3)
        e_i[i] = 1.0
        e_i_skew = skew_symmetric(e_i)
        dW_d_theta = e_i_skew @ W
        return dW_d_theta.flatten()
        
    dW_d_theta_x = dW_d_theta_i(0)
    dW_d_theta_y = dW_d_theta_i(1)
    dW_d_theta_z = dW_d_theta_i(2)
    
    # Construct the 9x6 matrix
    # Columns 0-2 (Translation): Zero
    # Columns 3-5 (Rotation): dW/d_theta_x, dW/d_theta_y, dW/d_theta_z
    d_W_d_T = np.zeros((9, 6))
    d_W_d_T[:, 3] = dW_d_theta_x
    d_W_d_T[:, 4] = dW_d_theta_y
    d_W_d_T[:, 5] = dW_d_theta_z
    
    return d_mu_C_d_T, d_W_d_T
```

**Parameters:**
*   **Iterations:** At least 50 iterations of gradient descent per frame are required for accurate tracking [S1, unofficial].
*   **Hyperparameters:** Photometric/Geometric loss weighting $\lambda_{pho}$ is used to balance $E_{pho}$ and $E_{geo}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The use of Lie algebra ensures minimal parameterization, avoiding singularities associated with Euler angles. However, numerical stability of the Exp/Log maps must be maintained for large perturbations [S1, unofficial].
*   **Boundaries:** The method assumes the Gaussian splatting rasterization is differentiable. Degenerate cases where Gaussians are far from the camera or have zero opacity may result in vanishing gradients [S1, unofficial].

## Subtopics

### Photometric and Geometric Tracking
This variant optimizes only the current camera pose without updating the map representation. It minimizes photometric and/or geometric residuals to track the camera motion. It is chosen when the scene geometry is static and the primary goal is accurate pose estimation rather than mapping. It requires rendering the Gaussians from the current pose and comparing against observed images or depth maps [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The analytical derivation eliminates the overhead of automatic differentiation, making the gradient computation significantly faster. This is crucial for real-time SLAM, where at least 50 iterations per frame are needed [S1, unofficial].
*   **Memory Complexity:** Minimal parameterization reduces the size of the Jacobian matrices, lowering memory bandwidth requirements during optimization [S1, unofficial].
*   **Performance:** The system operates at 3.2 FPS with monocular input and 2.5 FPS with depth input on the Replica dataset [S1, unofficial].
*   **Strengths:** Enables efficient camera pose optimization in 3DGS-based SLAM; provides the first analytical Jacobian for 3DGS splatting; ensures minimal parameterization [S1, unofficial].
*   **Limitations:** Requires accurate initial pose for convergence; performance depends on the number of Gaussians and resolution [S1, unofficial].

## Background
The method originates from the "Gaussian Splatting SLAM" paper, which introduces a SLAM system using 3D Gaussians as the sole representation. It addresses the limitation of prior methods that assumed fewer Gaussians and used ray-intersection, which are not applicable to the high-density 3DGS splatting pipeline. By deriving analytical Jacobians on the Lie group, it enables efficient tracking and mapping in dynamic environments [S1, unofficial].

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
