# 2D Gaussian Projection with Covariance Stabilization

Also called: 2D Gaussian Projection, 3D Gaussian Projection, 3D to 2D Projection in Gaussian Splatting, EWA Projection, Elliptical Weighted Average projection, Gaussian Projection, Linearized 2D Gaussian Projection, Splat Projection

## Overview
2D Gaussian Projection with Covariance Stabilization is the core rasterization primitive of 3D Gaussian Splatting that maps an anisotropic 3D Gaussian volume primitive into a 2D elliptical footprint (splat) on the image plane [S3, unofficial]. It solves the problem of rendering millions of 3D probability distributions efficiently by approximating the non-linear perspective projection with a local affine transformation, thereby preserving the Gaussian form in screen space [S3, unofficial]. It explicitly computes the screen-space covariance matrix to handle shape and spread, unlike point-sprite methods [S1, unofficial]. The method stabilizes the projection by adding a small epsilon to the diagonal of the 2D covariance matrix to guarantee positive definiteness and numerical stability [S2, unofficial].

## How it works
**Inputs:**
*   $M$ 3D Gaussians, each defined by:
    *   Mean position in world space $\mu_w \in \mathbb{R}^3$ [S9, unofficial].
    *   Rotation quaternion $q \in \mathbb{R}^4$ (normalized) [S8, unofficial].
    *   Scale vector $s \in \mathbb{R}^3$ (log-scale parameters) [S8, unofficial].
    *   Opacity $\alpha \in [0, 1]$ [S8, unofficial].
    *   Spherical Harmonic (SH) coefficients $h$ for view-dependent color [S7, unofficial].
*   Camera parameters:
    *   View matrix $V \in \mathbb{R}^{4 \times 4}$ (world-to-camera) [S1, unofficial].
    *   Projection matrix $P \in \mathbb{R}^{4 \times 4}$ (perspective) [S1, unofficial].
    *   Image dimensions $W, H$ (width, height in pixels) [S1, unofficial].
    *   Principal point $(c_x, c_y)$ (often center of image) [S1, unofficial].

**Outputs:**
*   A list of projected 2D Gaussians, each containing:
    *   Screen-space mean $\mu_{2D} \in \mathbb{R}^2$ [S1, unofficial].
    *   Screen-space covariance matrix $\Sigma_{2D} \in \mathbb{R}^{2 \times 2}$ [S2, unofficial].
    *   Inverse covariance (conic) matrix $\Sigma_{2D}^{-1} \in \mathbb{R}^{2 \times 2}$ [S2, unofficial].
    *   Depth $z_c$ (for sorting) [S2, unofficial].
    *   Bounding box (AABB) in screen coordinates [S2, unofficial].
    *   Color and opacity attributes [S8, unofficial].

**Data Structures:**
The projected Gaussian is stored in a struct with the following fields [S2, unofficial]:
*   `screen`: `Vec2` (pixel center $sx, sy$).
*   `depth`: `f32` ($z_c$, positive in front of camera).
*   `cov2d_inv`: `Mat2` (inverse 2D covariance).
*   `bbox`: `[i32; 4]` (inclusive $x_0, y_0, x_1, y_1$).
*   `color`: `Vec3` (RGB).
*   `opacity`: `f32` ($[0, 1]$).

**Steps:**

1.  **Reconstruct 3D Covariance Matrix:**
    Compute the $3 \times 3$ covariance matrix $\Sigma$ from the quaternion $q$ and scale $s$ [S1, unofficial].
    $$
    R = \text{QuaternionToMatrix}(q)
    $$
    $$
    S = \text{diag}(\exp(s_0), \exp(s_1), \exp(s_2))
    $$
    $$
    \Sigma = R S S^\top R^\top
    $$
    Note: $\Sigma$ is symmetric positive semi-definite [S1, unofficial].

2.  **Transform to Camera Space:**
    Transform the mean and covariance to camera space using the view matrix $V$ [S1, unofficial].
    $$
    \mu_c = V_{3 \times 3} \mu_w + V_{3 \times 1}
    $$
    $$
    \Sigma_c = V_{3 \times 3} \Sigma V_{3 \times 3}^\top
    $$
    Here, $V_{3 \times 3}$ is the upper-left $3 \times 3$ rotation block of the view matrix [S1, unofficial]. Translation is ignored for covariance as it describes spread around the centroid [S1, unofficial].

3.  **Compute Projection Jacobian:**
    Linearize the perspective projection at $\mu_c = (x_c, y_c, z_c)$ using the Jacobian $J$ [S1, unofficial]. The perspective projection maps $(x, y, z)$ to $(u, v)$:
    $$
    u = f_x \frac{x}{z} + c_x, \quad v = f_y \frac{y}{z} + c_y
    $$
    The Jacobian $J \in \mathbb{R}^{2 \times 3}$ evaluated at $\mu_c$ is:
    $$
    J = \begin{pmatrix}
    \frac{f_x}{z_c} & 0 & -\frac{f_x x_c}{z_c^2} \\
    0 & \frac{f_y}{z_c} & -\frac{f_y y_c}{z_c^2}
    \end{pmatrix}
    $$
    Focal lengths $f_x, f_y$ are derived from the projection matrix or camera intrinsics [S1, unofficial].

4.  **Project Covariance to 2D:**
    Compute the screen-space covariance $\Sigma_{2D}$ by applying the Jacobian to the camera-space covariance [S3, unofficial]:
    $$
    \Sigma_{2D} = J \Sigma_c J^\top
    $$
    This results in a $2 \times 2$ symmetric matrix [S3, unofficial].

5.  **Covariance Stabilization (Low-Pass Filter):**
    Add a small epsilon $\epsilon_{2d}$ to the diagonal of $\Sigma_{2D}$ to ensure positive definiteness and numerical stability [S2, unofficial].
    $$
    \Sigma_{2D}^{\text{stab}} = \Sigma_{2D} + \epsilon_{2d} I
    $$
    Typical value for $\epsilon_{2d}$ is $0.3$ [S2, unofficial][S5, unofficial]. Some implementations use $0.01$ [S4, unofficial]. This ensures each Gaussian spans at least the width of a single pixel [S5, unofficial].

6.  **Compute Screen-Space Mean:**
    Project the camera-space mean $\mu_c$ to screen coordinates [S1, unofficial]:
    $$
    \mu_{2D} = \left( f_x \frac{x_c}{z_c} + c_x, \quad f_y \frac{y_c}{z_c} + c_y \right)
    $$

7.  **Compute Inverse Covariance (Conic):**
    Compute the inverse of the stabilized covariance matrix $\Sigma_{2D}^{\text{stab}}$ for efficient evaluation in the fragment shader [S2, unofficial]. For a symmetric matrix $\begin{bmatrix} a & b \\ b & d \end{bmatrix}$:
    $$
    \det = ad - b^2
    $$
    If $\det \le 0$, cull the splat [S2, unofficial].
    $$
    \Sigma_{2D}^{-1} = \frac{1}{\det} \begin{bmatrix} d & -b \\ -b & a \end{bmatrix}
    $$

8.  **Compute Bounding Box:**
    Determine the axis-aligned bounding box (AABB) for the splat to optimize rasterization [S1, unofficial].
    Compute eigenvalues $\lambda_{1,2}$ of $\Sigma_{2D}^{\text{stab}}$:
    $$
    \lambda_{1,2} = \frac{\text{tr}(\Sigma_{2D}^{\text{stab}})}{2} \pm \sqrt{\frac{\text{tr}(\Sigma_{2D}^{\text{stab}})^2}{4} - \det(\Sigma_{2D}^{\text{stab}})}
    $$
    Compute unit eigenvectors $e_1, e_2$.
    Define half-axes $b_1 = 3\sqrt{\lambda_1}e_1$ and $b_2 = 3\sqrt{\lambda_2}e_2$ (using $3\sigma$ cutoff) [S1, unofficial].
    The bounding box is the AABB of the parallelogram spanned by $b_1$ and $b_2$ centered at $\mu_{2D}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def quaternion_to_rotation_matrix(q):
    """Converts a normalized quaternion (w, x, y, z) to a 3x3 rotation matrix."""
    w, x, y, z = q
    return np.array([
        [1 - 2*(y*y + z*z), 2*(x*y - z*w),     2*(x*z + y*w)],
        [2*(x*y + z*w),     1 - 2*(x*x + z*z), 2*(y*z - x*w)],
        [2*(x*z - y*w),     2*(y*z + x*w),     1 - 2*(x*x + y*y)]
    ])

def project_gaussian(gaussian, camera):
    """
    Projects a single 3D Gaussian to 2D screen space.
    
    Args:
        gaussian: dict with keys 'pos' (3,), 'rot' (4,), 'scale' (3,), 'opacity' (float)
        camera: dict with keys 'view_matrix' (4x4), 'proj_matrix' (4x4), 'fx', 'fy', 'cx', 'cy', 'width', 'height'
    
    Returns:
        dict with keys 'mu_2d', 'cov2d_inv', 'depth', 'bbox', 'color', 'opacity' or None if culled.
    """
    # 1. Reconstruct 3D Covariance
    R = quaternion_to_rotation_matrix(gaussian['rot'])
    S = np.diag(np.exp(gaussian['scale']))
    Sigma_3d = R @ S @ S.T @ R.T
    
    # 2. Transform to Camera Space
    V = camera['view_matrix']
    V_rot = V[:3, :3]
    V_trans = V[:3, 3]
    
    mu_c = V_rot @ gaussian['pos'] + V_trans
    Sigma_c = V_rot @ Sigma_3d @ V_rot.T
    
    x_c, y_c, z_c = mu_c
    
    # Cull if behind camera
    if z_c <= 0.0:
        return None

    # 3. Compute Jacobian
    fx = camera['fx']
    fy = camera['fy']
    J = np.array([
        [fx / z_c, 0, -fx * x_c / (z_c**2)],
        [0, fy / z_c, -fy * y_c / (z_c**2)]
    ])
    
    # 4. Project Covariance
    Sigma_2d = J @ Sigma_c @ J.T
    
    # 5. Stabilize Covariance
    eps_2d = 0.3 # Default stabilization factor
    Sigma_2d_stab = Sigma_2d + eps_2d * np.eye(2)
    
    # 6. Compute Screen Mean
    mu_2d = np.array([
        fx * (x_c / z_c) + camera['cx'],
        fy * (y_c / z_c) + camera['cy']
    ])
    
    # 7. Compute Inverse Covariance
    det = Sigma_2d_stab[0,0] * Sigma_2d_stab[1,1] - Sigma_2d_stab[0,1]**2
    if det <= 0.0:
        return None # Cull singular covariance
    
    inv_det = 1.0 / det
    cov2d_inv = np.array([
        [Sigma_2d_stab[1,1] * inv_det, -Sigma_2d_stab[0,1] * inv_det],
        [-Sigma_2d_stab[0,1] * inv_det, Sigma_2d_stab[0,0] * inv_det]
    ])
    
    # 8. Compute Bounding Box (3-sigma)
    eigenvalues, eigenvectors = np.linalg.eigh(Sigma_2d_stab)
    # eigenvalues are sorted ascending, so lambda1 is smallest
    lambda1, lambda2 = eigenvalues[0], eigenvalues[1]
    e1, e2 = eigenvectors[:, 0], eigenvectors[:, 1]
    
    b1 = 3.0 * np.sqrt(lambda1) * e1
    b2 = 3.0 * np.sqrt(lambda2) * e2
    
    # Bounding box corners
    corners = [
        mu_2d - b1 - b2,
        mu_2d + b1 - b2,
        mu_2d + b1 + b2,
        mu_2d - b1 + b2
    ]
    corners = np.array(corners)
    bbox = np.array([
        np.floor(np.min(corners[:, 0])),
        np.floor(np.min(corners[:, 1])),
        np.ceil(np.max(corners[:, 0])),
        np.ceil(np.max(corners[:, 1]))
    ])
    
    # Clip bbox to image bounds
    bbox[0] = max(0, bbox[0])
    bbox[1] = max(0, bbox[1])
    bbox[2] = min(camera['width'] - 1, bbox[2])
    bbox[3] = min(camera['height'] - 1, bbox[3])
    
    if bbox[0] > bbox[2] or bbox[1] > bbox[3]:
        return None

    # SH color evaluation not given in sources [S8, unofficial]
    color = np.array([1.0, 1.0, 1.0]) 
    
    return {
        'mu_2d': mu_2d,
        'cov2d_inv': cov2d_inv,
        'depth': z_c,
        'bbox': bbox,
        'color': color,
        'opacity': gaussian['opacity']
    }
```

**Edge Cases and Pitfalls:**
*   **Perspective Divide Non-Linearity:** The projection is non-linear [S3, unofficial]. The affine approximation is valid only locally around the centroid [S1, unofficial]. For very large Gaussians or extreme FOV, this approximation may introduce geometric warping artifacts [S3, unofficial].
*   **Singular Covariance:** If the determinant of the 2D covariance is zero or negative, the inverse is undefined [S2, unofficial]. This can happen if the Gaussian is viewed edge-on or if numerical errors accumulate [G]. Stabilization ($\epsilon_{2d}$) mitigates this [S2, unofficial].
*   **Depth Sorting:** Gaussians must be sorted by depth ($z_c$) before compositing to ensure correct alpha blending [S7, unofficial].
*   **Aliasing:** Without stabilization, small Gaussians can cause high-frequency aliasing. The low-pass filter ensures minimum pixel coverage [S5, unofficial].

## Subtopics

### [Adaptive Bounding Box Computation](1206-adaptive-bounding-box-computation.md)
Computes a tight, axis-aligned bounding box (AABB) for a 2D Gaussian splat based on an opacity threshold, replacing the loose circular bounding box used in original 3D Gaussian Splatting. It sets itself apart from siblings by explicitly integrating the splat's opacity and a configurable threshold to prune negligible contributions, thereby reducing false-positive tile intersections during rasterization.

### [Render Means](2070-render-means.md)
Render Means is the simplest possible rendering mode for 3D Gaussian Splatting, where each Gaussian is represented by a single pixel at the location where its 3D mean projects onto the 2D image plane. It demonstrates the basic camera transformation pipeline (world → camera → image) and frustum culling without any rasterization complexity. Unlike its sibling [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md), which computes screen-space covariance matrices for elliptical splatting, Render Means ignores shape and spread, treating Gaussians as point sprites.

### [Optimal Gaussian Splatting Projection](3311-optimal-gaussian-splatting-projection.md)
Optimal Gaussian Splatting Projection is a rendering method that minimizes projection error by projecting each 3D Gaussian onto a plane tangent to the line connecting the Gaussian mean to the camera center, rather than a fixed image plane. It sets itself apart from standard 3D-GS by using a unique tangent plane for each Gaussian based on its position relative to the camera, significantly reducing artifacts like blurriness and needle-like structures, particularly in wide-field-of-view or short focal length scenarios.

### [Projection Jacobian Computation](4272-projection-jacobian-computation.md)
Computes the $2 \times 3$ Jacobian matrix of the perspective projection function with respect to 3D coordinates, used for linearizing the projection of 3D Gaussians into screen space. It transforms the 3D covariance structure into 2D screen-space covariance, enabling accurate Gaussian splatting.

### [Per-Primitive Texture Mapping for Gaussian Splatting](8622-per-primitive-texture-mapping-for-gaussian-splatting.md)
Per-Primitive Texture Mapping for Gaussian Splatting assigns material properties as texture maps in the splat's local coordinate system rather than single attribute values per primitive, enabling higher-frequency spatial detail without increasing primitive count. It decouples geometry from appearance, allowing complex visual features and detailed normal maps to be captured within the existing Gaussian representation. This method differs from standard Gaussian Splatting by treating alpha-blending as a smoothing filter in a deferred rendering pipeline, which stabilizes the optimization of texture-sampled features and produces more cohesive results compared to shading directly on Gaussians.

### [3D Ellipsoid Projection](15723-3d-ellipsoid-projection.md)
3D Ellipsoid Projection is a Gaussian selection method used to identify candidate 3D Gaussians for rendering specific image slices in volume rendering pipelines. It approximates the spatial extent of a 3D Gaussian ellipsoid as a sphere to compute a uniform 2D bounding box, which is then projected orthogonally onto every image slice. Unlike sibling methods that adapt bounding boxes per slice or use complex covariance corrections, this approach prioritizes simplicity by treating the Gaussian's range as isotropic, though this leads to redundant computation for off-center slices.

## Operating characteristics
*   **Time Complexity:** $O(M)$ for projection of $M$ Gaussians, where each projection involves constant-time matrix operations ($3 \times 3$ and $2 \times 2$) [G]. Sorting takes $O(M \log M)$. Rasterization is $O(P)$ where $P$ is the number of pixels, but optimized by tile-based binning to only process relevant splats [S2, unofficial].
*   **Memory Complexity:** $O(M)$ to store the projected splats [G].
*   **Measured Performance:** The method enables real-time rendering of millions of Gaussians [S3, unofficial]. For example, on the Mip-NeRF360 benchmark, 3DGS reaches 27.21 dB PSNR and 0.214 LPIPS at 134 fps [S3, unofficial]. An accelerator implementation achieved 129 FPS at Full HD (1080p) [S12].
*   **Scaling:** Scales linearly with the number of Gaussians. Performance is heavily dependent on the efficiency of the sorting and tile-based rasterization stages [G].
*   **Strengths:** Closed-form projection allows for differentiable rendering, enabling gradient-based optimization of scene parameters [S3, unofficial]. High rendering speed due to efficient rasterization [S3, unofficial].
*   **Limitations:** The affine approximation of perspective projection can introduce artifacts for very large Gaussians or extreme viewpoints [S3, unofficial]. Requires careful handling of numerical stability (covariance stabilization) [S2, unofficial].
*   **When to use:** When real-time or differentiable rendering of large-scale 3D scenes is required [G].
*   **When not to use:** When high geometric accuracy for very large primitives is critical, or when memory bandwidth is severely constrained and $M$ is extremely large [G].

## Background
The method was originally described by Zwicker et al. in 2002 for anti-aliased splatting of volume data [S3, unofficial]. It was later reused by 3D Gaussian Splatting (3DGS) for the closed-form projection of 3D Gaussians, enabling real-time rendering of photorealistic scenes [S3, unofficial]. The technique is fundamental to the 3DGS rendering pipeline, allowing the conversion of 3D probability distributions into 2D splats for efficient rasterization [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S2] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S3] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S4] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S5] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S6] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S7] 驚くほどキレイな三次元シーン復元、「3D Gaussian Splatting」を徹底的に解説する, https://qiita.com/scomup/items/d5790da25a846e645de1, fetched 2026-10-29 (unofficial)
[S8] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
[S9] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
[S10] https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, https://github.com/KhronosGroup/glTF/blob/main/extensions/2.0/Khronos/KHR_gaussian_splatting/README.md, fetched 2026-10-30 (unofficial)
[S11] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
[S12] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
