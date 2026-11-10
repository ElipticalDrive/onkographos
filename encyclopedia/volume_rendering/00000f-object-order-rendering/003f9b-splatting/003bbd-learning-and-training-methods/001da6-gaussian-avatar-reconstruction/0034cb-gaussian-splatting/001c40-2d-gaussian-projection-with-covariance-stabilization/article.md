# 2D Gaussian Projection with Covariance Stabilization

Also called: 2D Gaussian Projection, 2D covariance transformation, EWA Projection, Elliptical Weighted Average projection, Gaussian Projection, Linearized 2D Gaussian Projection, Splat Projection

## Overview
This method projects a 3D Gaussian primitive, defined by a mean and a covariance matrix in world space, into a 2D elliptical footprint (splat) on the image plane for rasterization [S1, S3, unofficial]. It solves the problem of rendering millions of anisotropic 3D Gaussians in real-time by avoiding ray marching; instead, it approximates the non-linear perspective projection with a local affine transformation, preserving the Gaussian form in 2D [S3, unofficial]. It sets itself apart from [Optimal Gaussian Splatting Projection](3311-optimal-gaussian-splatting-projection.md) by using a fixed image plane rather than a per-Gaussian tangent plane, and from [2D Gaussian Covariance Stabilization and Rescaling](3467-2d-gaussian-covariance-stabilization-and-rescaling.md) by adding a constant epsilon to the diagonal for positive definiteness rather than rescaling the determinant [S2, S3467].

## How it works
**Inputs:**
*   $M$ 3D Gaussians, each defined by:
    *   Mean $\mu_w \in \mathbb{R}^3$ (world coordinates).
    *   Covariance $\Sigma_w \in \mathbb{R}^{3 \times 3}$ (symmetric positive semi-definite), often stored as a rotation quaternion $q$ and scale vector $s$ where $\Sigma_w = R S S^\top R^\top$ [S1, S7, unofficial].
    *   Opacity $\alpha \in [0, 1]$.
    *   Color coefficients (e.g., Spherical Harmonics) [S7, unofficial].
*   Camera parameters: View matrix $V$, Projection matrix $P$, image width $W$, height $H$, focal lengths $f_x, f_y$, principal point $c_x, c_y$ [S1, S7, unofficial].

**Outputs:**
*   For each Gaussian, a 2D Gaussian splat defined by:
    *   Screen mean $\mu_{2D} \in \mathbb{R}^2$.
    *   2D Covariance $\Sigma_{2D} \in \mathbb{R}^{2 \times 2}$.
    *   Inverse 2D Covariance (Conic) $\Sigma_{2D}^{-1} \in \mathbb{R}^{2 \times 2}$ [S2, unofficial].
    *   Bounding Box (AABB) in pixel coordinates [S2, unofficial].

**Data Structures:**
The projected splat is typically stored in a struct containing:
*   `screen`: `Vec2` (pixel center $sx, sy$) [S2, unofficial].
*   `depth`: `f32` (camera-space $z_c$, positive in front) [S2, unofficial].
*   `cov2d_inv`: `Mat2` (inverse 2D covariance) [S2, unofficial].
*   `bbox`: `[i32; 4]` (inclusive $x_0, y_0, x_1, y_1$) [S2, unofficial].
*   `color`: `Vec3` (RGB) [S2, unofficial].
*   `opacity`: `f32` [S2, unofficial].

**Steps:**

1.  **Reconstruct 3D Covariance:** If not already provided, compute $\Sigma_w$ from quaternion $q$ and scale $s$.
    $$ R = \text{QuaternionToMatrix}(q) $$
    $$ S = \text{diag}(\exp(s_0), \exp(s_1), \exp(s_2)) $$
    $$ \Sigma_w = R S S^\top R^\top $$
    [S1, S7, unofficial]

2.  **Transform to Camera Space:**
    Transform the mean to camera space:
    $$ \mu_c = V_{3\times3} \mu_w + V_{3\times1} $$
    Transform the covariance to camera space (rotation only, as translation does not affect spread):
    $$ \Sigma_c = V_{3\times3} \Sigma_w V_{3\times3}^\top $$
    [S1, S7, unofficial]

3.  **Compute Projection Jacobian:**
    Evaluate the Jacobian $J$ of the perspective projection at $\mu_c = (x_c, y_c, z_c)$:
    $$ J = \begin{pmatrix} -\frac{f_x}{z_c} & 0 & \frac{f_x x_c}{z_c^2} \\ 0 & -\frac{f_y}{z_c} & \frac{f_y y_c}{z_c^2} \end{pmatrix} $$
    where $f_x, f_y$ are focal lengths in pixels [S1, unofficial]. Note: Some sources define $J$ with positive signs for the diagonal terms depending on coordinate system conventions (e.g., right-handed vs left-handed, or $z$ direction), but the structure remains the same [S3, S7, unofficial].
    [S1, S3, S7, unofficial]

4.  **Project Covariance:**
    Compute the 2D covariance matrix:
    $$ \Sigma_{2D} = J \Sigma_c J^\top $$
    Extract the top-left $2 \times 2$ block [S1, S3, unofficial].

5.  **Covariance Stabilization (Low-Pass Filter):**
    Add a small constant $\epsilon_{2d}$ to the diagonal elements of $\Sigma_{2D}$ to ensure positive definiteness and prevent aliasing when the Gaussian projects to sub-pixel sizes:
    $$ \Sigma_{2D} \leftarrow \Sigma_{2D} + \epsilon_{2d} I $$
    Typical value: $\epsilon_{2d} = 0.3$ [S2, S5, unofficial]. This ensures each Gaussian covers at least approximately 1 pixel [S5, unofficial].

6.  **Compute Screen Mean:**
    Project the camera-space mean to screen coordinates:
    $$ u = f_x \frac{x_c}{z_c} + c_x $$
    $$ v = f_y \frac{y_c}{z_c} + c_y $$
    [S2, S7, unofficial]

7.  **Compute Inverse Covariance (Conic):**
    Compute the inverse of the stabilized $\Sigma_{2D} = \begin{bmatrix} a & b \\ b & d \end{bmatrix}$:
    $$ \det = ad - b^2 $$
    If $\det \le 0$, cull the splat [S2, unofficial].
    $$ \Sigma_{2D}^{-1} = \frac{1}{\det} \begin{bmatrix} d & -b \\ -b & a \end{bmatrix} $$
    [S2, unofficial]

8.  **Compute Bounding Box:**
    Compute eigenvalues $\lambda_{1,2}$ of $\Sigma_{2D}$:
    $$ \lambda_{1,2} = \frac{\text{tr}(\Sigma_{2D})}{2} \pm \sqrt{\frac{\text{tr}(\Sigma_{2D})^2}{4} - \det(\Sigma_{2D})} $$
    Compute unit eigenvectors $e_1, e_2$.
    Define half-axes $b_1 = 3\sqrt{\lambda_1}e_1$ and $b_2 = 3\sqrt{\lambda_2}e_2$ (using $3\sigma$ cutoff) [S1, unofficial].
    Compute the AABB in pixel coordinates based on $\mu_{2D}$ and $b_1, b_2$ [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def project_gaussian(mu_w, cov_w, view_matrix, proj_matrix, width, height, fov_y, aspect_ratio):
    """
    Projects a 3D Gaussian to 2D screen space.
    
    Args:
        mu_w: (3,) array, world space mean.
        cov_w: (3, 3) array, world space covariance.
        view_matrix: (4, 4) array, view matrix.
        proj_matrix: (4, 4) array, projection matrix.
        width: int, image width.
        height: int, image height.
        fov_y: float, vertical field of view in radians.
        aspect_ratio: float, width / height.
        
    Returns:
        dict with keys: 'mu_2d', 'cov_2d', 'cov_2d_inv', 'bbox', 'depth'
    """
    # 1. Transform to camera space
    # View matrix V: [R | t]
    R_cam = view_matrix[:3, :3]
    t_cam = view_matrix[:3, 3]
    
    mu_c = R_cam @ mu_w + t_cam
    cov_c = R_cam @ cov_w @ R_cam.T
    
    x_c, y_c, z_c = mu_c
    
    # Cull if behind camera
    if z_c <= 0.0:
        return None

    # 2. Compute Focal Lengths
    # f_y = height / (2 * tan(fov_y / 2))
    # f_x = f_y * aspect_ratio
    f_y = height / (2.0 * math.tan(fov_y / 2.0))
    f_x = f_y * aspect_ratio
    
    # Principal point (assuming centered)
    c_x = width / 2.0
    c_y = height / 2.0

    # 3. Compute Jacobian J
    # J = [ -fx/z, 0,    fx*x/z^2 ]
    #     [ 0,    -fy/z, fy*y/z^2 ]
    inv_z = 1.0 / z_c
    inv_z_sq = inv_z * inv_z
    
    J = np.array([
        [-f_x * inv_z, 0.0, f_x * x_c * inv_z_sq],
        [0.0, -f_y * inv_z, f_y * y_c * inv_z_sq]
    ])

    # 4. Project Covariance
    # Sigma_2D = J @ Sigma_c @ J.T
    cov_2d = J @ cov_c @ J.T
    
    # 5. Stabilization (Low-pass filter)
    eps_2d = 0.3
    cov_2d[0, 0] += eps_2d
    cov_2d[1, 1] += eps_2d

    # 6. Compute Screen Mean
    mu_2d = np.array([
        f_x * (x_c / z_c) + c_x,
        f_y * (y_c / z_c) + c_y
    ])

    # 7. Compute Inverse Covariance (Conic)
    det = cov_2d[0, 0] * cov_2d[1, 1] - cov_2d[0, 1] * cov_2d[0, 1]
    if det <= 0.0:
        return None # Cull splat
    
    inv_det = 1.0 / det
    cov_2d_inv = np.array([
        [cov_2d[1, 1] * inv_det, -cov_2d[0, 1] * inv_det],
        [-cov_2d[0, 1] * inv_det, cov_2d[0, 0] * inv_det]
    ])

    # 8. Compute Bounding Box (3-sigma)
    # Eigenvalues of 2x2 symmetric matrix
    trace = cov_2d[0, 0] + cov_2d[1, 1]
    det_cov = cov_2d[0, 0] * cov_2d[1, 1] - cov_2d[0, 1]**2
    
    # Discriminant
    disc = (trace**2 / 4.0) - det_cov
    if disc < 0:
        disc = 0.0 # Numerical stability
    
    sqrt_disc = math.sqrt(disc)
    lambda_1 = trace / 2.0 + sqrt_disc
    lambda_2 = trace / 2.0 - sqrt_disc
    
    # Eigenvectors
    # e1: solve (Sigma - lambda1*I)v = 0
    # v1 = (Sigma[0,1], lambda1 - Sigma[0,0])
    v1 = np.array([cov_2d[0, 1], lambda_1 - cov_2d[0, 0]])
    norm_v1 = np.linalg.norm(v1)
    if norm_v1 == 0:
        e1 = np.array([1.0, 0.0])
    else:
        e1 = v1 / norm_v1
    
    # e2 is orthogonal to e1
    e2 = np.array([-e1[1], e1[0]])
    
    # Half axes
    b1 = 3.0 * math.sqrt(lambda_1) * e1
    b2 = 3.0 * math.sqrt(lambda_2) * e2
    
    # Bounding box corners
    # Center is mu_2d
    # Corners are mu_2d +/- b1 +/- b2
    # We need the min/max x and y
    offsets = [
        b1 + b2,
        b1 - b2,
        -b1 + b2,
        -b1 - b2
    ]
    
    min_x = mu_2d[0]
    max_x = mu_2d[0]
    min_y = mu_2d[1]
    max_y = mu_2d[1]
    
    for offset in offsets:
        x = mu_2d[0] + offset[0]
        y = mu_2d[1] + offset[1]
        if x < min_x: min_x = x
        if x > max_x: max_x = x
        if y < min_y: min_y = y
        if y > max_y: max_y = y
        
    # Convert to integer pixel coordinates (inclusive)
    bbox = [
        int(math.floor(min_x)),
        int(math.floor(min_y)),
        int(math.ceil(max_x)),
        int(math.ceil(max_y))
    ]
    
    return {
        'mu_2d': mu_2d,
        'cov_2d': cov_2d,
        'cov_2d_inv': cov_2d_inv,
        'bbox': bbox,
        'depth': z_c
    }
```

## Subtopics

### [Adaptive Bounding Box Computation](1206-adaptive-bounding-box-computation.md)
Computes a tight AABB for a 2D Gaussian splat based on an opacity threshold, replacing the loose circular bounding box used in original 3DGS. It sets itself apart from siblings like [Tight Ellipse AABB Derivation](2826-tight-ellipse-aabb-derivation.md) by explicitly integrating the splat's opacity and a configurable threshold to prune negligible contributions, thereby reducing false-positive tile intersections during rasterization.

### [Render Means](2070-render-means.md)
Render Means is the simplest possible rendering mode for 3D Gaussian Splatting, where each Gaussian is represented by a single pixel at the location where its 3D mean projects onto the 2D image plane. It demonstrates the basic camera transformation pipeline (world → camera → image) and frustum culling without any rasterization complexity. Unlike its sibling [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md), which computes screen-space covariance matrices for elliptical splatting, Render Means ignores shape and spread, treating Gaussians as point sprites.

### [Optimal Gaussian Splatting Projection](3311-optimal-gaussian-splatting-projection.md)
Optimal Gaussian Splatting Projection is a rendering method that minimizes projection error by projecting each 3D Gaussian onto a plane tangent to the line connecting the Gaussian mean to the camera center, rather than a fixed image plane. It sets itself apart from standard 3D-GS by using a unique tangent plane for each Gaussian based on its position relative to the camera, significantly reducing artifacts like blurriness and needle-like structures, particularly in wide-field-of-view or short focal length scenarios.

### [2D Gaussian Covariance Stabilization and Rescaling](3467-2d-gaussian-covariance-stabilization-and-rescaling.md)
This method stabilizes the projection of 3D Gaussians into screen space by applying a low-pass filter to the 2D covariance matrix, ensuring that each Gaussian spans at least the width of a single pixel to reduce aliasing artifacts. It computes the inverse covariance (conic) matrix and a scalar rescale factor that preserves the integrated density of the Gaussian after filtering. It differs from its parent, [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md), which adds a constant epsilon to guarantee positive definiteness, by explicitly scaling the determinant to maintain energy conservation relative to the filtered covariance.

### [Projection Jacobian Computation](4272-projection-jacobian-computation.md)
Computes the $2 \times 3$ Jacobian matrix of the perspective projection function with respect to 3D coordinates, used for linearizing the projection of 3D Gaussians into screen space. It transforms the 3D covariance structure into 2D screen-space covariance, enabling accurate Gaussian splatting.

### [Per-Primitive Texture Mapping for Gaussian Splatting](8622-per-primitive-texture-mapping-for-gaussian-splatting.md)
Per-Primitive Texture Mapping for Gaussian Splatting assigns material properties as texture maps in the splat's local coordinate system rather than single attribute values per primitive, enabling higher-frequency spatial detail without increasing primitive count. It decouples geometry from appearance, allowing complex visual features and detailed normal maps to be captured within the existing Gaussian representation. This method differs from standard Gaussian Splatting by treating alpha-blending as a smoothing filter in a deferred rendering pipeline, which stabilizes the optimization of texture-sampled features and produces more cohesive results compared to shading directly on Gaussians.

### [2D Covariance Correction via Linear Regression](9912-2d-covariance-correction-via-linear-regression.md)
This method approximates the relationship between projected 2D covariances and actual 2D covariances using a linear regression model, serving as a computationally efficient alternative to neural network-based corrections in Gaussian Avatar Reconstruction. It sets itself apart from its siblings by replacing complex learned mappings with closed-form linear coefficients or a single scalar factor, thereby reducing training overhead while maintaining reconstruction quality comparable to Half-Cosine (HC) kernels.

### Ellipse Parameter Extraction
Extracts the semi-major/minor axes, rotation angle, and center position from a 2D covariance matrix (represented as a 3x3 matrix with the third row/col being 0 or ignored) [S4, unofficial].
*   **Step:** Compute semi-major axis: $\sqrt{0.5 \cdot (\text{cov}[0,0] + \text{cov}[1,1] + \sqrt{(\text{cov}[0,0]-\text{cov}[1,1])^2 + 4\cdot\text{cov}[0,1]^2})}$. Compute semi-minor axis: $\sqrt{0.5 \cdot (\text{cov}[0,0] + \text{cov}[1,1] - \sqrt{(\text{cov}[0,0]-\text{cov}[1,1])^2 + 4\cdot\text{cov}[0,1]^2})}$. Compute rotation components $j$ (sin) and $k$ (cos) using half-angle identities on the covariance matrix elements. Adjust signs of $j$ and $k$ based on the sign of $\text{cov}[0,1]$ and $(\text{cov}[0,0]-\text{cov}[1,1])$. Swap $j$ and $k$ if $\text{cov}[0,0] - \text{cov}[1,1] < 0$. Compute the center (translation) by solving the linear system derived from the partial derivatives of the implicit ellipse equation [S4, unofficial].
*   **Pitfall:** If the discriminant ($\text{cov}[0,0]\cdot\text{cov}[1,1] - \text{cov}[0,1]^2$) is zero, the inverse is undefined. The code does not explicitly check for this, implying it assumes valid positive-definite covariance matrices [S4, unofficial].

### Covariance Correction Factor
A scalar correction factor applied to the covariance matrix to approximate the relationship between projected 2D covariances and actual 2D covariances, replacing a more complex MLP or Simpson's rule integration [S10].
*   **Parameters:** Inverse Multiquadric correction factor: 1.38f. Parabolic correction factor: 1.3f [S10].

### Skip Zero-Jacobian Matrix Multiplication
An optimization in the projection stage that skips multiplications involving zero elements in the Jacobian matrix, reducing computation by 52.53% compared to the original 3DGS projection step [S11]. Skips projection-side computation for Gaussians whose Jacobian determinant is zero or negligible, avoiding unnecessary math [S11].
*   **Step:** Computes or checks the Jacobian for each Gaussian; if zero, skips the projection and blending steps for that point [S11].
*   **Performance:** Increases throughput from 35.94% to 75.70% (2.11× gain), raising FPS from 46.4 to 97.7 at 1080p [S11].

## Operating characteristics
*   **Time Complexity:** $O(M)$ for projection of $M$ Gaussians, assuming constant-time matrix operations. Sorting is $O(M \log M)$ or $O(M)$ with bucket sort. Rasterization is $O(N \cdot K)$ where $N$ is pixels and $K$ is average splats per pixel [S3, unofficial].
*   **Memory Complexity:** $O(M)$ to store projected splats [S2, unofficial].
*   **Performance:** Enables real-time rendering (e.g., 134 fps on Mip-NeRF360) by avoiding ray marching [S3, unofficial].
*   **Strengths:** Closed-form projection allows differentiability and fast rasterization. The $3\sigma$ bounding box provides a tight fit for culling [S1, unofficial].
*   **Limitations:** The local affine approximation introduces geometric warping errors, particularly for large Gaussians or extreme FOV [S3, unofficial]. Sub-pixel Gaussians can cause aliasing without stabilization [S5, unofficial].
*   **When to use:** Real-time rendering of large-scale 3D scenes from calibrated images.
*   **When not to use:** When high geometric accuracy for large, distant primitives is required (consider [Optimal Gaussian Splatting Projection](3311-optimal-gaussian-splatting-projection.md)).

## Background
Originally described by Zwicker et al. in 2002 for anti-aliased splatting [S3, unofficial]. Reused by 3DGS for the closed-form projection of 3D Gaussians [S3, unofficial]. The method relies on the property that a linear transformation of a Gaussian distribution is also a Gaussian distribution, allowing the non-linear perspective projection to be approximated locally by its Jacobian [S1, S3, unofficial].

## Sources
[S1] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S2] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S3] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S4] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S5] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S6] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S7] 驚くほどキレイな三次元シーン復元、「3D Gaussian Splatting」を徹底的に解説する, https://qiita.com/scomup/items/d5790da25a846e645de1, fetched 2026-10-29 (unofficial)
[S8] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
[S9] https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, fetched 2026-10-30
[S10] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
[S11] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
