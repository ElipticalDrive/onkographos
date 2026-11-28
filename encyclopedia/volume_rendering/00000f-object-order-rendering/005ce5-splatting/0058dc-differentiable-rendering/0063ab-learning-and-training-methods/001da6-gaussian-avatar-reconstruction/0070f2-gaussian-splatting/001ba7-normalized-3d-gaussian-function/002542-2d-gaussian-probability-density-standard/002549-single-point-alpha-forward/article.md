# Single Point Alpha Forward

Also called: Gaussian Alpha Computation, Single Point Alpha

## Overview
Computes the alpha value for a single 3D Gaussian projected onto a 2D pixel by evaluating the 2D Gaussian probability density function (PDF) at the pixel center [S1, unofficial]. It builds on [2D Gaussian Probability Density (Standard)](3466-2d-gaussian-probability-density-standard.md) by providing the full pipeline to transform 3D Gaussian parameters into the 2D screen-space covariance required for that evaluation. It differs from [2D Gaussian Probability Density (Conic)](3475-2d-gaussian-probability-density-conic.md) by explicitly computing the inverse covariance matrix for the PDF evaluation rather than using conic coefficients, and from [Gradient of 2D Gaussian PDF (Conic)](3472-gradient-of-2d-gaussian-pdf-conic.md) by returning a scalar alpha value rather than its gradient.

## How it works
**Inputs:**
- $xyz$: 3D position of the Gaussian center (world space).
- $q$: Quaternion representing the Gaussian orientation (w, x, y, z).
- $s$: Scale vector (3 elements), typically stored as log-scale.
- $\sigma$: Opacity parameter (scalar, in range [0, 1]).
- $T_{camera}$: $4 \times 4$ world-to-camera transformation matrix.
- $K$: $3 \times 3$ camera intrinsic matrix.
- $uv_{center}$: 2D coordinates of the pixel center in screen space.

**Outputs:**
- $\alpha$: Scalar alpha value for the Gaussian at the given pixel.

**Data Structures:**
- Matrices are $3 \times 3$ or $4 \times 4$ standard floating-point arrays.
- Vectors are standard 3D or 4D floating-point arrays.

**Steps:**

1. **Homogenize Position:** Create a 4D homogeneous vector from the 3D position.
   $$xyz1 = [xyz_x, xyz_y, xyz_z, 1]^T$$

2. **Transform to Camera Space:** Apply the world-to-camera transformation.
   $$xyz1_{camera} = T_{camera} \cdot xyz1$$

3. **Extract Camera Coordinates:** Take the first three components of the transformed vector.
   $$xyz_{camera} = [xyz1_{camera, x}, xyz1_{camera, y}, xyz1_{camera, z}]^T$$

4. **Project to Screen Space:**
   - Compute homogeneous screen coordinates:
     $$uv1 = K \cdot xyz_{camera}$$
   - Perform perspective division to get 2D screen coordinates:
     $$uv = \left[ \frac{uv1_x}{uv1_z}, \frac{uv1_y}{uv1_z} \right]^T$$

5. **Compute Scale Matrix:**
   - Exponentiate the scale vector to get linear scales:
     $$S_{linear} = \exp(s)$$
   - Construct the diagonal scale matrix:
     $$S = \text{diag}(S_{linear})$$

6. **Compute Rotation Matrix:**
   - Convert quaternion $q$ to a $3 \times 3$ rotation matrix $R$.

7. **Compute 3D Covariance:**
   - Construct the 3D covariance matrix $\Sigma$:
     $$\Sigma = R S S^T R^T$$
   - Note: Since $S$ is diagonal, $S S^T = S^2$.

8. **Compute Projection Jacobian:**
   - Compute the Jacobian $J$ of the perspective projection. For a point at depth $z = xyz_{camera, z}$, the $2 \times 3$ Jacobian is:
     $$J = \begin{bmatrix} \frac{K_{00}}{z} & 0 & -\frac{K_{00} \cdot xyz_{camera, x}}{z^2} \\ 0 & \frac{K_{11}}{z} & -\frac{K_{11} \cdot xyz_{camera, y}}{z^2} \end{bmatrix}$$
   - Where $K_{00}$ and $K_{11}$ are the focal length components ($f_x, f_y$) of the intrinsic matrix $K$.

9. **Compute 2D Covariance:**
   - Extract the upper-left $3 \times 3$ rotation part of $T_{camera}$, denoted as $W$.
   - Compute the 2D covariance matrix $cov$:
     $$cov = J W \Sigma W^T J^T$$

10. **Compute Probability Density:**
    - Compute the inverse of the 2D covariance matrix $cov^{-1}$.
    - Calculate the squared Mahalanobis distance:
      $$d^2 = (uv_{center} - uv)^T \cdot cov^{-1} \cdot (uv_{center} - uv)$$
    - Evaluate the unnormalized Gaussian PDF:
      $$p = \exp\left(-0.5 \cdot d^2\right)$$

11. **Compute Alpha:**
    - Combine the opacity and the PDF value:
      $$\alpha = \sigma \cdot p$$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def single_point_alpha_forward(xyz, q, s, sigma, T_camera, K, uv_center):
    """
    Computes the alpha value for a single 3D Gaussian at a specific pixel.
    
    Args:
        xyz: np.ndarray, shape (3,), world space position.
        q: np.ndarray, shape (4,), quaternion (w, x, y, z).
        s: np.ndarray, shape (3,), log-scale.
        sigma: float, opacity.
        T_camera: np.ndarray, shape (4, 4), world-to-camera matrix.
        K: np.ndarray, shape (3, 3), camera intrinsics.
        uv_center: np.ndarray, shape (2,), pixel center coordinates.
        
    Returns:
        float: alpha value.
    """
    # 1. Homogenize position
    xyz1 = np.array([xyz[0], xyz[1], xyz[2], 1.0])
    
    # 2. Transform to camera space
    xyz1_camera = T_camera @ xyz1
    
    # 3. Extract camera coordinates
    xyz_camera = xyz1_camera[:3]
    z = xyz_camera[2]
    
    # 4. Project to UV
    uv1 = K @ xyz_camera
    uv = np.array([uv1[0] / uv1[2], uv1[1] / uv1[2]])
    
    # 5. Compute scale matrix S
    S_linear = np.exp(s)
    S = np.diag(S_linear)
    
    # 6. Compute rotation matrix R from quaternion q
    # q = [w, x, y, z]
    w, x, y, z_q = q[0], q[1], q[2], q[3]
    R = np.array([
        [1 - 2*(y**2 + z_q**2), 2*(x*y - w*z_q), 2*(x*z_q + w*y)],
        [2*(x*y + w*z_q), 1 - 2*(x**2 + z_q**2), 2*(y*z_q - w*x)],
        [2*(x*z_q - w*y), 2*(y*z_q + w*x), 1 - 2*(x**2 + y**2)]
    ])
    
    # 7. Compute 3D covariance Sigma
    # Sigma = R * S * S^T * R^T
    # Since S is diagonal, S * S^T = S^2
    S2 = S @ S
    Sigma = R @ S2 @ R.T
    
    # 8. Compute Jacobian J of projection
    # J is 2x3
    fx = K[0, 0]
    fy = K[1, 1]
    J = np.array([
        [fx / z, 0, -fx * xyz_camera[0] / (z**2)],
        [0, fy / z, -fy * xyz_camera[1] / (z**2)]
    ])
    
    # 9. Compute 2D covariance cov
    # W is the upper-left 3x3 part of T_camera
    W = T_camera[:3, :3]
    cov = J @ W @ Sigma @ W.T @ J.T
    
    # 10. Compute probability density p
    # Inverse of 2x2 matrix cov
    det = cov[0, 0] * cov[1, 1] - cov[0, 1] * cov[1, 0]
    if det == 0:
        # Handle degenerate case, e.g., return 0 or small epsilon
        return 0.0
    cov_inv = np.array([
        [cov[1, 1] / det, -cov[0, 1] / det],
        [-cov[1, 0] / det, cov[0, 0] / det]
    ])
    
    diff = uv_center - uv
    d2 = diff @ cov_inv @ diff
    p = np.exp(-0.5 * d2)
    
    # 11. Compute alpha
    alpha = sigma * p
    
    return alpha
```

**Parameters:**
- $\sigma$: Opacity, typically in $[0, 1]$.
- $s$: Log-scale, values depend on the Gaussian size.
- $q$: Unit quaternion.

**Edge Cases and Pitfalls:**
- **Degenerate Covariance:** If the determinant of $cov$ is zero (e.g., Gaussian viewed edge-on or at infinity), the inverse is undefined. A guard should return 0 or a small epsilon [G].
- **Depth $z \le 0$:** If the Gaussian is behind the camera, the projection is invalid. The method assumes valid input where $z > 0$ [G].
- **Precision:** Floating-point precision issues may arise in the matrix inversion for very small or large Gaussians.

## Subtopics

### [2D Gaussian Alpha Computation (Batched)](4273-2d-gaussian-alpha-computation-batched.md)
This method computes the alpha values for all $N$ 2D Gaussians across all $W \times H$ pixels in a single batched operation, serving as the core rasterization step in Gaussian Splatting. It builds on [Single Point Alpha Forward](3478-single-point-alpha-forward.md) by vectorizing the evaluation over the entire image grid, enabling efficient GPU execution [G]. It differs from [Explicit Intersection Alpha Compositing](4814-explicit-intersection-alpha-compositing.md) by computing raw alpha contributions for all pixel-gaussian pairs rather than identifying explicit intersection pairs for compositing.

### [3D Gaussian Point Spread Function Model](5086-3d-gaussian-point-spread-function-model.md)
The 3D Gaussian Point Spread Function (PSF) Model defines the spatial response of an imaging system using a 3D Gaussian kernel parameterized by a covariance matrix. It serves as the baseline differentiable model for blur in volumetric rendering, where the covariance matrix is optimized during inference to control the level of blur in the rendered image.

### [RoofGS Range-Aware Fast Exponential Evaluation](9821-roofgs-range-aware-fast-exponential-evaluation.md)
This method replaces the transcendental exponential function in the Gaussian blending loop with a range-aware bit-level approximation, leveraging the bounded exponent range resulting from opacity culling. It reduces special-function-unit (SFU) pressure and arithmetic overhead, achieving significant throughput gains with negligible visual quality loss.

## Operating characteristics
- **Time Complexity:** $O(1)$ per Gaussian-pixel pair.
- **Memory Complexity:** $O(1)$ auxiliary memory.
- **Strengths:** Simple, differentiable, and efficient for single-point evaluation.
- **Limitations:** Does not handle batch operations or culling; requires explicit pixel-Gaussian intersection checks in a full rasterizer.

## Background
Originated in the context of 3D Gaussian Splatting for differentiable rendering [S1, unofficial]. It is a fundamental component of the rasterization pipeline, enabling the computation of alpha values for compositing.

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
