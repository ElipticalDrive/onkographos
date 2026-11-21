# Single Point Alpha Forward

Also called: Gaussian Alpha Computation, Single Point Alpha

## Overview
Computes the alpha value for a single 3D Gaussian projected onto a 2D pixel [S1, unofficial]. It determines the opacity contribution of a single Gaussian splat to a pixel during object-order rasterization [G]. It differs from [2D Gaussian Probability Density (Standard)](3466-2d-gaussian-probability-density-standard.md) by including the full projection pipeline (camera transform, Jacobian approximation) and the final scaling by intrinsic opacity, whereas the parent method only evaluates the PDF value given a pre-computed 2D covariance [G].

## How it works
**Inputs:**
- $xyz \in \mathbb{R}^3$: World-space position of the Gaussian center [S1, unofficial].
- $q \in \mathbb{R}^4$: Quaternion representing the Gaussian orientation (w, x, y, z) [S1, unofficial].
- $s \in \mathbb{R}^3$: Log-scale vector (exponentiated to get scale) [S1, unofficial].
- $\sigma \in \mathbb{R}$: Intrinsic opacity parameter [S1, unofficial].
- $T_{camera} \in \mathbb{R}^{4 \times 4}$: World-to-camera transformation matrix [G].
- $K \in \mathbb{R}^{3 \times 3}$: Camera intrinsic matrix [G].
- $uv_{center} \in \mathbb{R}^2$: Screen-space coordinates of the pixel center [S1, unofficial].

**Outputs:**
- $\alpha \in \mathbb{R}$: The computed alpha value for this Gaussian at this pixel [S1, unofficial].

**Data Structures:**
- Matrices are stored in row-major or column-major format consistent with the linear algebra library used [G]. $T_{camera}$ is a $4 \times 4$ homogeneous transformation [G]. $K$ is a $3 \times 3$ intrinsic matrix [G].

**Steps:**

1.  **Homogenize Position:**
    Construct the homogeneous coordinate vector:
    $$ xyz1 = [x, y, z, 1]^T $$ [S1, unofficial]

2.  **Transform to Camera Space:**
    Apply the world-to-camera transformation:
    $$ xyz1_{camera} = T_{camera} \cdot xyz1 $$ [S1, unofficial]
    Extract the camera-space position $\mathbf{p}_{cam} = [x_c, y_c, z_c]^T$ from the first three components of $\mathbf{p}_{cam\_hom}$ [S1, unofficial].

3.  **Compute 3D Covariance Matrix:**
    Compute the scale matrix $S$ and rotation matrix $R$:
    $$ S = \text{diag}(\exp(s_x), \exp(s_y), \exp(s_z)) $$ [S1, unofficial]
    $$ R = \text{QuaternionToRotationMatrix}(q) $$ [S1, unofficial]
    Compute the 3D covariance matrix $\Sigma$:
    $$ \Sigma = R S S R^T $$ [S1, unofficial]

4.  **Compute Projection Jacobian:**
    Let $W$ be the upper-left $3 \times 3$ submatrix of $T_{camera}$ (the rotation part) [S1, unofficial].
    The projection from 3D camera space to 2D screen space is approximated by the Jacobian $J$ [S1, unofficial]. For a pinhole camera with intrinsics $K$, the Jacobian $J$ relating small changes in 3D camera coordinates to 2D screen coordinates is derived from the projection equations [G].
    $$ J = \frac{\partial (u, v)}{\partial (x_c, y_c, z_c)} $$ [G]
    Using the standard perspective projection $u = f_x \frac{x_c}{z_c} + c_x$ and $v = f_y \frac{y_c}{z_c} + c_y$:
    $$ J = \begin{bmatrix} \frac{f_x}{z_c} & 0 & -\frac{f_x x_c}{z_c^2} \\ 0 & \frac{f_y}{z_c} & -\frac{f_y y_c}{z_c^2} \end{bmatrix} $$ [G]
    *Note: Some implementations combine the rotation $W$ and the Jacobian $J$ into a single $2 \times 3$ matrix $J W$ to transform the 3D covariance directly [G].*

5.  **Compute 2D Covariance:**
    Transform the 3D covariance $\Sigma$ to 2D screen space covariance $cov$:
    $$ cov = J W \Sigma W^T J^T $$ [S1, unofficial]
    Here, $W$ rotates the covariance from world space to camera space (if $\Sigma$ was in world space) or is part of the projection chain [S1, unofficial]. In the standard Gaussian Splatting pipeline, $\Sigma$ is often defined in world space, so $W$ (the rotation part of $T_{camera}$) transforms it to camera space before projection [G]. If $\Sigma$ is already in camera space, $W$ is identity [G]. The source notes specify $W$ is the rotation part of $T_{camera}$, implying $\Sigma$ is in world space [S1, unofficial].

6.  **Compute Probability Density:**
    Calculate the Mahalanobis distance squared between the pixel center $uv_{center}$ and the projected Gaussian center $uv_{proj}$ [S1, unofficial].
    First, compute the projected center $uv_{proj}$:
    $$ uv1 = K \cdot xyz_{camera} $$ [S1, unofficial]
    $$ uv_{proj} = uv1_{0:2} / uv1_2 $$ [S1, unofficial]
    *Correction based on Step 4/5 logic:* The Jacobian $J$ is evaluated at $\mathbf{p}_{cam}$ [S1, unofficial]. The center of the Gaussian in screen space is $uv_{proj}$ [S1, unofficial].
    Let $\mathbf{d} = uv_{center} - uv_{proj}$ [S1, unofficial].
    Compute the inverse of the 2D covariance matrix $cov^{-1}$ [G].
    $$ p = \exp\left( -0.5 \cdot \mathbf{d}^T \cdot cov^{-1} \cdot \mathbf{d} \right) $$ [S1, unofficial]
    *Note: The normalization factor $\frac{1}{2\pi \sqrt{\det(cov)}}$ is omitted in the alpha computation step as it is absorbed into the opacity parameter or handled separately in the compositing equation [G]. The source note explicitly defines $p$ as the exponential term [S1, unofficial].*

7.  **Compute Alpha:**
    Scale the probability density by the intrinsic opacity $\sigma$:
    $$ \alpha = \sigma \cdot p $$ [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def single_point_alpha_forward(xyz, q, s, sigma, T_camera, K, uv_center):
    """
    Computes the alpha value for a single 3D Gaussian projected onto a 2D pixel.
    
    Args:
        xyz: np.array([x, y, z]), world space position.
        q: np.array([w, x, y, z]), quaternion orientation.
        s: np.array([sx, sy, sz]), log-scale.
        sigma: float, intrinsic opacity parameter.
        T_camera: np.array(4x4), world-to-camera transformation matrix.
        K: np.array(3x3), camera intrinsic matrix.
        uv_center: np.array([u, v]), screen space pixel center.
        
    Returns:
        alpha: float, computed alpha value.
    """
    # 1. Homogenize point position
    p_hom = np.array([xyz[0], xyz[1], xyz[2], 1.0])
    
    # 2. Transform to camera space
    p_cam_hom = T_camera @ p_hom
    p_cam = p_cam_hom[:3] # [x_c, y_c, z_c]
    z_c = p_cam[2]
    
    # 3. Compute 3D covariance Sigma
    # Scale matrix S
    scale = np.exp(s)
    S = np.diag(scale)
    
    # Rotation matrix R from quaternion q
    R = quaternion_to_rotation_matrix(q)
    
    # Sigma = R * S * S * R^T
    S2 = np.diag(scale**2)
    Sigma = R @ S2 @ R.T
    
    # 4. Compute Jacobian J of projection
    # W is the rotation part of T_camera (upper-left 3x3)
    W = T_camera[:3, :3]
    
    # Intrinsic matrix K
    fx = K[0, 0]
    fy = K[1, 1]
    cx = K[0, 2]
    cy = K[1, 2]
    
    # Projected center in screen space
    uv_proj = np.array([
        fx * p_cam[0] / z_c + cx,
        fy * p_cam[1] / z_c + cy
    ])
    
    # Jacobian J (2x3)
    J = np.array([
        [fx / z_c, 0.0, -fx * p_cam[0] / (z_c**2)],
        [0.0, fy / z_c, -fy * p_cam[1] / (z_c**2)]
    ])
    
    # 5. Compute 2D covariance cov
    # cov = J * W * Sigma * W^T * J^T
    Sigma_cam = W @ Sigma @ W.T
    cov = J @ Sigma_cam @ J.T
    
    # 6. Compute probability density p
    d = uv_center - uv_proj
    
    # Inverse of 2x2 covariance matrix
    det_cov = cov[0, 0] * cov[1, 1] - cov[0, 1] * cov[1, 0]
    if det_cov <= 0:
        return 0.0
    
    inv_cov = np.array([
        [cov[1, 1] / det_cov, -cov[0, 1] / det_cov],
        [-cov[1, 0] / det_cov, cov[0, 0] / det_cov]
    ])
    
    # Mahalanobis distance squared
    mahal_sq = d @ inv_cov @ d
    
    # Probability density (unnormalized)
    p = np.exp(-0.5 * mahal_sq)
    
    # 7. Compute alpha
    alpha = sigma * p
    
    return alpha

def quaternion_to_rotation_matrix(q):
    """
    Converts a quaternion [w, x, y, z] to a 3x3 rotation matrix.
    """
    w, x, y, z = q[0], q[1], q[2], q[3]
    
    # Normalize quaternion
    norm = np.sqrt(w**2 + x**2 + y**2 + z**2)
    w, x, y, z = w/norm, x/norm, y/norm, z/norm
    
    R = np.array([
        [1 - 2*(y**2 + z**2), 2*(x*y - w*z), 2*(x*z + w*y)],
        [2*(x*y + w*z), 1 - 2*(x**2 + z**2), 2*(y*z - w*x)],
        [2*(x*z - w*y), 2*(y*z + w*x), 1 - 2*(x**2 + y**2)]
    ])
    return R
```

**Parameters:**
- $\sigma$: Intrinsic opacity. Typical values depend on the training data, but often range from 0 to 1 after sigmoid activation, or are logits before activation [G].
- $s$: Log-scale. Typical values depend on the scene scale [G].

**Edge Cases and Pitfalls:**
- **Degenerate Covariance:** If the determinant of $cov$ is zero or negative (due to numerical error or extreme scaling), the inverse is undefined. Guard against this by checking the determinant and returning 0 or a small epsilon [G].
- **Z-depth:** If $z_c \le 0$, the Gaussian is behind the camera. The projection is invalid. Return 0 [G].
- **Precision:** The exponential function can underflow to 0 for large Mahalanobis distances. This is acceptable as it means the Gaussian does not contribute to the pixel [G].

## Subtopics

### [2D Gaussian Alpha Computation (Batched)](4273-2d-gaussian-alpha-computation-batched.md)
This method computes the alpha values for all $N$ 2D Gaussians across all $W \times H$ pixels in a single batched operation, serving as the core rasterization step in Gaussian Splatting. It builds on [Single Point Alpha Forward](3478-single-point-alpha-forward.md) by vectorizing the evaluation over the entire image grid, enabling efficient GPU execution [G]. It differs from [Explicit Intersection Alpha Compositing](4814-explicit-intersection-alpha-compositing.md) by computing raw alpha contributions for all pixel-gaussian pairs rather than identifying explicit intersection pairs for compositing.

### [RoofGS Range-Aware Fast Exponential Evaluation](9821-roofgs-range-aware-fast-exponential-evaluation.md)
This method replaces the transcendental exponential function in the Gaussian blending loop with a range-aware bit-level approximation, leveraging the bounded exponent range resulting from opacity culling. It sets itself apart from standard implementations by reducing special-function-unit (SFU) pressure and arithmetic overhead, achieving significant throughput gains with negligible visual quality loss. It builds on [Single Point Alpha Forward](3478-single-point-alpha-forward.md) by optimizing the final opacity evaluation step.

## Operating characteristics
- **Time Complexity:** $O(1)$ per Gaussian-pixel pair. The operations involve matrix-vector multiplications and a single exponential evaluation [G].
- **Memory Complexity:** $O(1)$ auxiliary memory per evaluation [G].
- **Performance:** The method is computationally lightweight per pixel but is called for every Gaussian-pixel intersection. In a full rasterization pass, the cost is proportional to the number of splats per pixel [G].
- **Strengths:** Simple, differentiable, and efficient for sparse scenes [G].
- **Limitations:** Does not handle occlusion or compositing order; these are handled in subsequent steps. The approximation of the projection using the Jacobian can introduce artifacts for large Gaussians or extreme viewing angles [G].

## Background
The method is a core component of 3D Gaussian Splatting, a technique for real-time radiance field rendering. It was introduced in the context of learning 3D Gaussian primitives from images. The specific formulation of projecting 3D Gaussians to 2D and computing alpha values is detailed in the implementation of Gaussian Splatting [S1, unofficial]. The underlying mathematical model of using a Gaussian kernel as a Point Spread Function (PSF) is also discussed in the context of differentiable rendering for fluorescence microscopy [S2, unofficial].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S2] Differentiable Rendering for 3D Fluorescence Microscopy, https://doi.org/10.48550/arxiv.2303.10440, fetched 2026-10-27 (unofficial)
