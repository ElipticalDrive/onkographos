# Child Gaussian Initialization

Also called: AdpSplit child initialization, Error-driven child creation

## Overview
Child Gaussian Initialization is the specific procedure for generating the geometric and photometric properties of new 3D Gaussian primitives during the splitting phase of adaptive densification [S1, unofficial]. It solves the problem of how to accurately map a parent Gaussian's error region—identified in camera space—into a valid 3D Gaussian in world space, ensuring the child is positioned on the scene geometry and oriented correctly relative to the viewing direction [S1, unofficial]. It sets itself apart from generic splitting methods by explicitly unprojecting image-space covariance ellipses into world-space scales and orientations using Mahalanobis distance minimization and perspective-aware clamping, rather than simple isotropic scaling or random sampling [S1, unofficial].

## How it works
**Inputs:**
*   Parent Gaussian $g_i$ with mean $\mu_i \in \mathbb{R}^3$, covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$, and opacity $\alpha_i$.
*   Camera intrinsics $(f_x, f_y, p_x, p_y)$ and extrinsics (camera center $o_v$ in world coordinates, camera-to-world rotation $R_{c2w}$).
*   Image-space error region centroid $(\bar{x}_{qv}, \bar{y}_{qv})$ and covariance matrix in image space (defined by PCA directions $e_{1}, e_{2}$ and standard deviations $\sigma_1, \sigma_2$).
*   Ground-truth RGB value $r_{qv}$ at the region centroid.
*   Parent maximum scale $s_{pmax} = \max_k(s_k)$ for clamping limits.

**Outputs:**
*   Child Gaussian $g_{qv}$ defined by mean $\mu_{qv}$, covariance $\Sigma_{qv}$, opacity $\alpha_{qv}$, and spherical harmonic coefficients $\mathbf{c}_{qv}$.

**Data Structures:**
*   The covariance $\Sigma$ is stored as a $3 \times 3$ symmetric positive-definite matrix, or equivalently parameterized by scale vector $s \in \mathbb{R}^3$ and rotation quaternion $q \in \mathbb{R}^4$ (or rotation matrix $R \in \mathbb{R}^{3 \times 3}$). In this method, the output is explicitly constructed as $R_{qv} \text{diag}(s_{qv}^2) R_{qv}^T$.

**Steps:**

1.  **Compute Camera-Space Ray Direction:**
    Calculate the unnormalized ray direction in camera space from the image centroid:
    $$
    d_{cam,v} = \begin{bmatrix} (\bar{x}_{qv} - p_x)/f_x \\ (\bar{y}_{qv} - p_y)/f_y \\ 1 \end{bmatrix}
    $$
    [S1, unofficial]

2.  **Transform to World-Space Ray:**
    Convert the ray direction to world coordinates and normalize it:
    $$
    \hat{d}_{qv} = \frac{R_{c2w} d_{cam,v}}{\| R_{c2w} d_{cam,v} \|}
    $$
    [S1, unofficial]

3.  **Define Back-Projected Ray:**
    Define the ray $\ell_{qv}(t)$ originating from the camera center $o_v$:
    $$
    \ell_{qv}(t) = o_v + t \hat{d}_{qv}
    $$
    [S1, unofficial]

4.  **Find Optimal Depth ($t^*$):**
    Find the point on the ray that minimizes the Mahalanobis distance to the parent center $\mu_i$. Let $A = \Sigma_i^{-1}$ and $b = \mu_i - o_v$. The distance squared is $f(t) = (t \hat{d}_{qv} - b)^T A (t \hat{d}_{qv} - b)$. Minimizing $f(t)$ yields:
    $$
    t^* = \frac{b^T A \hat{d}_{qv}}{\hat{d}_{qv}^T A \hat{d}_{qv} + \epsilon}
    $$
    where $\epsilon$ is a small constant (e.g., $10^{-8}$) added to the denominator for numerical stability [S1, unofficial].

5.  **Set Child Mean:**
    The child's 3D position is the point on the ray at depth $t^*$:
    $$
    \mu_{qv} = \ell_{qv}(t^*)
    $$
    [S1, unofficial]

6.  **Compute Camera-Space Depth:**
    Calculate the depth in camera space units:
    $$
    t^*_{z,v} = \frac{t^*}{\| d_{cam,v} \|}
    $$
    [S1, unofficial]

7.  **Unproject Image-Space Covariance to World-Space Axes:**
    Let $u_v$ and $w_v$ be the camera right and down directions in world coordinates (derived from $R_{c2w}$). The image-space PCA directions $e_{1}, e_{2}$ and scales $\sigma_1, \sigma_2$ define the error ellipse. We compute intermediate components $w_{1x,v}, w_{1y,v}, w_{2x,v}, w_{2y,v}$ using a clamping function $\text{clip}(x, a) = \text{sign}(x) \min(|x|, a)$ to prevent excessively large scales:
    $$
    w_{1x,v} = \text{clip}\left( \frac{t^*_{z,v}}{f_x} e_{1,x} \sigma_1, s_{pmax} |e_{1,x}| \right)
    $$
    $$
    w_{1y,v} = \text{clip}\left( \frac{t^*_{z,v}}{f_y} e_{1,y} \sigma_1, s_{pmax} |e_{1,y}| \right)
    $$
    $$
    w_{2x,v} = \text{clip}\left( \frac{t^*_{z,v}}{f_x} e_{2,x} \sigma_2, s_{pmax} |e_{2,x}| \right)
    $$
    $$
    w_{2y,v} = \text{clip}\left( \frac{t^*_{z,v}}{f_y} e_{2,y} \sigma_2, s_{pmax} |e_{2,y}| \right)
    $$
    [S1, unofficial]

    Unproject these components into world-space vectors $a_{q1,v}$ and $a_{q2,v}$:
    $$
    a_{q1,v} = w_{1x,v} u_v + w_{1y,v} w_v
    $$
    $$
    a_{q2,v} = w_{2x,v} u_v + w_{2y,v} w_v
    $$
    [S1, unofficial]

8.  **Determine Child Scales:**
    The scales are the lengths of the unprojected vectors. The third scale (depth) is set equal to the second scale (transverse):
    $$
    s_{q1,v} = \| a_{q1,v} \|, \quad s_{q2,v} = \| a_{q2,v} \|, \quad s_{q3,v} = s_{q2,v}
    $$
    [S1, unofficial]

9.  **Enforce Orthonormality:**
    Due to perspective projection and clamping, $a_{q1,v}$ and $a_{q2,v}$ may not be orthogonal. Apply Gram-Schmidt orthogonalization to obtain orthonormal in-plane axes $\hat{u}_1, \hat{u}_2$:
    $$
    \hat{u}_1 = \frac{a_{q1,v}}{\| a_{q1,v} \|}
    $$
    $$
    \hat{u}_2 = \frac{a_{q2,v} - (a_{q2,v}^T \hat{u}_1)\hat{u}_1}{\| a_{q2,v} - (a_{q2,v}^T \hat{u}_1)\hat{u}_1 \|}
    $$
    [S1, unofficial]

10. **Construct Child Covariance:**
    Let $z_v$ be the camera forward direction in world coordinates. Form the rotation matrix $R_{qv} = [\hat{u}_1 \mid \hat{u}_2 \mid z_v]$. The child covariance is:
    $$
    \Sigma_{qv} = R_{qv} \text{diag}(s_{q1,v}^2, s_{q2,v}^2, s_{q3,v}^2) R_{qv}^T
    $$
    [S1, unofficial]

11. **Initialize Color and Opacity:**
    *   **Color:** Set the DC spherical harmonic coefficient to the ground-truth RGB value $r_{qv}$ at the centroid. Set all higher-order SH coefficients to zero [S1, unofficial].
    *   **Opacity:** Set $\alpha_{qv} = \alpha_i$ (inherit parent opacity) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_child_gaussian(parent_gaussian, camera_intrinsics, camera_extrinsics, 
                              image_region, ground_truth_rgb, parent_max_scale):
    """
    Initialize a child Gaussian from an error region in a specific view.
    
    Args:
        parent_gaussian: dict with 'mu' (3,), 'Sigma' (3,3), 'alpha' (float)
        camera_intrinsics: dict with 'fx', 'fy', 'px', 'py'
        camera_extrinsics: dict with 'ov' (3,), 'Rc2w' (3,3)
        image_region: dict with 'centroid' (2,), 'e1' (2,), 'e2' (2,), 'sigma1', 'sigma2'
        ground_truth_rgb: (3,) array
        parent_max_scale: float
        
    Returns:
        child_gaussian: dict with 'mu', 'Sigma', 'alpha', 'SH_dc'
    """
    # 1. Camera-space ray direction
    cx, cy = image_region['centroid']
    fx, fy, px, py = camera_intrinsics['fx'], camera_intrinsics['fy'], \
                     camera_intrinsics['px'], camera_intrinsics['py']
    
    d_cam = np.array([(cx - px) / fx, (cy - py) / fy, 1.0])
    
    # 2. World-space unit ray direction
    Rc2w = camera_extrinsics['Rc2w']
    d_world_unnorm = Rc2w @ d_cam
    d_hat = d_world_unnorm / np.linalg.norm(d_world_unnorm)
    
    # 3. Ray parameters
    ov = camera_extrinsics['ov']
    mu_i = parent_gaussian['mu']
    Sigma_i_inv = np.linalg.inv(parent_gaussian['Sigma'])
    
    # 4. Optimal depth t*
    A = Sigma_i_inv
    b = mu_i - ov
    
    numerator = b @ A @ d_hat
    denominator = d_hat @ A @ d_hat + 1e-8 # epsilon for stability
    t_star = numerator / denominator
    
    # 5. Child mean
    mu_child = ov + t_star * d_hat
    
    # 6. Camera-space depth
    t_z = t_star / np.linalg.norm(d_cam)
    
    # 7. Unproject covariance
    # Camera axes in world space
    u_v = Rc2w @ np.array([1.0, 0.0, 0.0]) # Right
    w_v = Rc2w @ np.array([0.0, 1.0, 0.0]) # Down
    
    e1 = image_region['e1']
    e2 = image_region['e2']
    s1 = image_region['sigma1']
    s2 = image_region['sigma2']
    
    def clip(x, a):
        return np.sign(x) * np.minimum(np.abs(x), a)
    
    # Compute components
    w1x = clip((t_z / fx) * e1[0] * s1, parent_max_scale * np.abs(e1[0]))
    w1y = clip((t_z / fy) * e1[1] * s1, parent_max_scale * np.abs(e1[1]))
    w2x = clip((t_z / fx) * e2[0] * s2, parent_max_scale * np.abs(e2[0]))
    w2y = clip((t_z / fy) * e2[1] * s2, parent_max_scale * np.abs(e2[1]))
    
    # Unproject to world vectors
    a1 = w1x * u_v + w1y * w_v
    a2 = w2x * u_v + w2y * w_v
    
    # 8. Child scales
    s1_child = np.linalg.norm(a1)
    s2_child = np.linalg.norm(a2)
    s3_child = s2_child
    
    # 9. Orthonormalization (Gram-Schmidt)
    u1_hat = a1 / s1_child
    u2_hat = a2 - np.dot(a2, u1_hat) * u1_hat
    u2_hat = u2_hat / np.linalg.norm(u2_hat)
    
    # 10. Construct Covariance
    z_v = Rc2w @ np.array([0.0, 0.0, 1.0]) # Forward
    R_child = np.column_stack((u1_hat, u2_hat, z_v))
    
    scales_sq = np.array([s1_child**2, s2_child**2, s3_child**2])
    Sigma_child = R_child @ np.diag(scales_sq) @ R_child.T
    
    # 11. Color and Opacity
    alpha_child = parent_gaussian['alpha']
    SH_dc = ground_truth_rgb # Assuming SH_dc is just RGB for DC term
    
    return {
        'mu': mu_child,
        'Sigma': Sigma_child,
        'alpha': alpha_child,
        'SH_dc': SH_dc
    }
```

**Edge Cases and Pitfalls:**
*   **Numerical Stability:** The denominator in $t^*$ calculation can approach zero if the ray is nearly orthogonal to the principal axis of the parent covariance. Adding $\epsilon$ is required [S1, unofficial].
*   **Orthogonality Loss:** Perspective projection and clamping can break the orthogonality of the unprojected axes. Gram-Schmidt is mandatory to ensure a valid rotation matrix [S1, unofficial].
*   **Scale Explosion:** Without clamping using $s_{pmax}$, unprojected scales can become arbitrarily large in the distance, leading to rendering artifacts. The clamping function limits the scale relative to the parent's maximum extent [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per child Gaussian, involving matrix inversions ($3 \times 3$) and vector operations.
*   **Memory Complexity:** $O(1)$ additional memory per child.
*   **Strengths:** Produces geometrically accurate child positions on the scene surface (via Mahalanobis minimization) and correctly oriented scales that respect perspective distortion. This leads to faster convergence and better detail preservation compared to isotropic splitting.
*   **Limitations:** Computationally more expensive than simple random sampling or isotropic scaling. Requires access to camera intrinsics/extrinsics and image-space error regions.
*   **When to use:** When high-fidelity geometry reconstruction is required and the parent Gaussian has identified a significant error region in a specific view.

## Background
This method was introduced in the **AdpSplit** paper, which proposes error-driven adaptive splitting for 3D Gaussian Splatting [S1, unofficial]. It addresses the limitations of standard densification strategies that do not account for the specific geometry of the error region or the perspective projection effects when creating new primitives. The method is part of the broader effort to optimize the initialization of new Gaussians during the training process to accelerate convergence and improve final rendering quality.

## Sources
[S1] AdpSplit: Error-Driven Adaptive Splitting for Faster Geometry Discovery in 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2605.06876, fetched 2026-10-29 (unofficial)
