# 2D Gaussian Projection with Covariance Stabilization

Also called: 2D Gaussian Projection, 2D covariance transformation, EWA Projection, Elliptical Weighted Average projection, Gaussian Projection, Linearized 2D Gaussian Projection

## Overview
This method projects a 3D Gaussian primitive, defined by a mean and a covariance matrix in world space, onto the 2D image plane to create a screen-space Gaussian splat [S3, unofficial]. It solves the problem of rendering millions of 3D Gaussians efficiently by avoiding ray marching; instead, it approximates the non-linear perspective projection with a local affine transformation, preserving the Gaussian form as a 2D ellipse [S1, S3, S4, unofficial]. It sets itself apart from naive point-splatting by accounting for anisotropic spread and orientation, and from exact perspective projection by using a first-order Taylor expansion (Jacobian) to maintain computational tractability and differentiability [S3, S5, unofficial].

## How it works
**Inputs:**
- 3D Gaussian parameters: mean $\mu \in \mathbb{R}^3$, rotation quaternion $q$, scale vector $s$, opacity $\alpha$, spherical harmonic coefficients $h$ [S7, unofficial].
- Camera parameters: View matrix $V$, Projection matrix $P$, image dimensions $W \times H$, focal lengths $f_x, f_y$, principal point $c_x, c_y$ [S7, S8, unofficial].

**Outputs:**
- Projected 2D mean $\mu_{2D} \in \mathbb{R}^2$ (pixel coordinates) [S1, unofficial].
- Projected 2D covariance $\Sigma_{2D} \in \mathbb{R}^{2 \times 2}$ [S1, unofficial].
- Inverse 2D covariance (conic) $\Sigma_{2D}^{-1}$ [S2, unofficial].
- Depth $z_c$ for sorting [S2, unofficial].
- Bounding box for rasterization [S1, unofficial].

**Data Structures:**
The projected Gaussian is typically stored in a struct containing:
- `screen`: `Vec2` (pixel center $s_x, s_y$) [S2, unofficial].
- `depth`: `f32` (camera-space $z_c$) [S2, unofficial].
- `cov2d_inv`: `Mat2` (inverse 2D covariance) [S2, unofficial].
- `bbox`: `[i32; 4]` (inclusive $x_0, y_0, x_1, y_1$) [S2, unofficial].
- `color`: `Vec3` (RGB) [S2, unofficial].
- `opacity`: `f32` ($[0, 1]$) [S2, unofficial].

**Steps:**

1. **Reconstruct 3D Covariance:**
   Convert the parameterized rotation $q$ and scale $s$ into a $3 \times 3$ covariance matrix $\Sigma$ [S7, unofficial].
   $$ R = \text{QuaternionToMatrix}(q) $$
   $$ S = \text{diag}(\exp(s_0), \exp(s_1), \exp(s_2)) $$
   $$ \Sigma = R S S^\top R^\top $$
   [S1, S7, unofficial]

2. **Transform to Camera Space:**
   Transform the covariance from world space to camera (eye) space using the rotation block of the view matrix $V$ [S1, unofficial]. Translation is ignored for covariance [S1, unofficial].
   $$ M = V_{3 \times 3} $$
   $$ \Sigma_{cam} = M \Sigma M^\top $$
   Transform the mean to camera space:
   $$ \mu_c = V \cdot [\mu, 1]^\top $$
   [S1, S7, unofficial]

3. **Compute Projection Jacobian:**
   Linearize the perspective projection at $\mu_c = (x_c, y_c, z_c)$ [S1, unofficial]. The projection function maps 3D camera coordinates to 2D pixel coordinates [S5, unofficial].
   $$ J = \begin{pmatrix} -\frac{f_x}{z_c} & 0 & \frac{f_x x_c}{z_c^2} \\ 0 & -\frac{f_y}{z_c} & \frac{f_y y_c}{z_c^2} \end{pmatrix} $$
   [S1, unofficial]
   *Note: Some implementations define $J$ with positive signs for the diagonal terms depending on coordinate system handedness, while others use negative signs [S1, S3, unofficial].*

4. **Project Covariance to 2D:**
   Apply the Jacobian to the camera-space covariance to obtain the screen-space covariance [S3, unofficial].
   $$ \Sigma_{2D} = J \Sigma_{cam} J^\top $$
   Extract the top-left $2 \times 2$ block [S1, unofficial].
   [S1, S3, S5, unofficial]

5. **Covariance Stabilization (Low-Pass Filter):**
   Add a small constant $\epsilon_{2d}$ to the diagonal elements of $\Sigma_{2D}$ to ensure positive definiteness and prevent aliasing/undersampling [S2, S5, unofficial].
   $$ \Sigma_{2D}[0,0] \leftarrow \Sigma_{2D}[0,0] + \epsilon_{2d} $$
   $$ \Sigma_{2D}[1,1] \leftarrow \Sigma_{2D}[1,1] + \epsilon_{2d} $$
   Typical value: $\epsilon_{2d} = 0.3$ [S2, S5, unofficial].

6. **Compute Inverse Covariance (Conic):**
   Compute the inverse of the stabilized $2 \times 2$ matrix analytically [S2, unofficial].
   $$ \det = \Sigma_{2D}[0,0]\Sigma_{2D}[1,1] - \Sigma_{2D}[0,1]^2 $$
   If $\det \le 0$, cull the splat [S2, unofficial].
   $$ \Sigma_{2D}^{-1} = \frac{1}{\det} \begin{bmatrix} \Sigma_{2D}[1,1] & -\Sigma_{2D}[0,1] \\ -\Sigma_{2D}[1,0] & \Sigma_{2D}[0,0] \end{bmatrix} $$
   [S2, unofficial]

7. **Compute 2D Mean:**
   Project the 3D mean to screen coordinates [S5, unofficial].
   $$ \mu_{2D} = \left( f_x \frac{x_c}{z_c} + c_x, \quad f_y \frac{y_c}{z_c} + c_y \right) $$
   [S2, S7, unofficial]

8. **Compute Bounding Box:**
   Calculate the eigenvalues $\lambda_{1,2}$ and eigenvectors $e_1, e_2$ of $\Sigma_{2D}$ [S1, unofficial].
   $$ \lambda_{1,2} = \frac{\text{tr}(\Sigma_{2D})}{2} \pm \sqrt{\frac{\text{tr}(\Sigma_{2D})^2}{4} - \det(\Sigma_{2D})} $$
   Define half-axes using a $3\sigma$ cutoff:
   $$ b_1 = 3\sqrt{\lambda_1} e_1, \quad b_2 = 3\sqrt{\lambda_2} e_2 $$
   The bounding box is the axis-aligned box enclosing the parallelogram spanned by $b_1$ and $b_2$ centered at $\mu_{2D}$ [S1, unofficial].
   [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def project_gaussian(mean_3d, quat, scale, view_matrix, proj_matrix, focal_x, focal_y, cx, cy, width, height):
    """
    Projects a 3D Gaussian to 2D screen space with covariance stabilization.
    
    Args:
        mean_3d: (3,) array, world space mean.
        quat: (4,) array, quaternion (w, x, y, z).
        scale: (3,) array, log-scale.
        view_matrix: (4,4) array, world-to-camera transform.
        proj_matrix: (4,4) array, camera-to-clip transform.
        focal_x, focal_y: float, focal lengths in pixels.
        cx, cy: float, principal point.
        width, height: int, image dimensions.
        
    Returns:
        dict with 'mu_2d', 'cov2d', 'cov2d_inv', 'depth', 'bbox', 'cull'
    """
    # 1. Reconstruct 3D Covariance
    R = quaternion_to_matrix(quat)
    S_diag = [math.exp(s) for s in scale]
    S = [[S_diag[0], 0, 0], [0, S_diag[1], 0], [0, 0, S_diag[2]]]
    
    # Sigma = R * S * S^T * R^T
    RS = mat_mul(R, S)
    Sigma = mat_mul(RS, transpose(RS))
    
    # 2. Transform to Camera Space
    # Extract 3x3 rotation block from view matrix
    M = [[view_matrix[i][j] for j in range(3)] for i in range(3)]
    
    # Sigma_cam = M * Sigma * M^T
    Sigma_cam = mat_mul(mat_mul(M, Sigma), transpose(M))
    
    # Mean in camera space
    mean_4d = [mean_3d[0], mean_3d[1], mean_3d[2], 1.0]
    mean_cam_4d = mat_mul_vec(view_matrix, mean_4d)
    x_c, y_c, z_c = mean_cam_4d[0], mean_cam_4d[1], mean_cam_4d[2]
    
    # Cull if behind camera
    if z_c <= 0.0:
        return {'cull': True}
        
    # 3. Compute Jacobian
    # J = [[-fx/z, 0, fx*x/z^2], [0, -fy/z, fy*y/z^2]]
    inv_z = 1.0 / z_c
    inv_z_sq = inv_z * inv_z
    J = [
        [-focal_x * inv_z, 0.0, focal_x * x_c * inv_z_sq],
        [0.0, -focal_y * inv_z, focal_y * y_c * inv_z_sq]
    ]
    
    # 4. Project Covariance
    # Sigma_2d = J * Sigma_cam * J^T
    A = mat_mul(J, Sigma_cam)
    Sigma_2d = mat_mul(A, transpose(J))
    
    # 5. Stabilization
    eps = 0.3
    Sigma_2d[0][0] += eps
    Sigma_2d[1][1] += eps
    
    # 6. Inverse Covariance
    det = Sigma_2d[0][0] * Sigma_2d[1][1] - Sigma_2d[0][1] * Sigma_2d[1][0]
    if det <= 0.0:
        return {'cull': True}
        
    inv_det = 1.0 / det
    cov2d_inv = [
        [Sigma_2d[1][1] * inv_det, -Sigma_2d[0][1] * inv_det],
        [-Sigma_2d[1][0] * inv_det, Sigma_2d[0][0] * inv_det]
    ]
    
    # 7. 2D Mean
    mu_2d_x = focal_x * (x_c * inv_z) + cx
    mu_2d_y = focal_y * (y_c * inv_z) + cy
    
    # 8. Bounding Box (3-sigma)
    # Eigenvalues of 2x2 symmetric matrix
    tr = Sigma_2d[0][0] + Sigma_2d[1][1]
    det_val = Sigma_2d[0][0] * Sigma_2d[1][1] - Sigma_2d[0][1]**2
    half_tr = tr / 2.0
    delta = math.sqrt(max(0.0, half_tr**2 - det_val))
    lambda1 = half_tr + delta
    lambda2 = half_tr - delta
    
    # Eigenvectors
    # e1 for lambda1
    if abs(Sigma_2d[0][1]) > 1e-6:
        v1_x = Sigma_2d[0][1]
        v1_y = lambda1 - Sigma_2d[0][0]
    else:
        v1_x = 1.0
        v1_y = 0.0
        
    norm_v1 = math.sqrt(v1_x**2 + v1_y**2)
    e1_x = v1_x / norm_v1
    e1_y = v1_y / norm_v1
    
    # e2 orthogonal to e1
    e2_x = -e1_y
    e2_y = e1_x
    
    # Half axes
    b1_x = 3.0 * math.sqrt(lambda1) * e1_x
    b1_y = 3.0 * math.sqrt(lambda1) * e1_y
    b2_x = 3.0 * math.sqrt(lambda2) * e2_x
    b2_y = 3.0 * math.sqrt(lambda2) * e2_y
    
    # Bounding box corners
    # The quad is spanned by b1 and b2.
    # Corners are mu +/- b1 +/- b2
    corners = [
        [mu_2d_x + b1_x + b2_x, mu_2d_y + b1_y + b2_y],
        [mu_2d_x + b1_x - b2_x, mu_2d_y + b1_y - b2_y],
        [mu_2d_x - b1_x + b2_x, mu_2d_y - b1_y + b2_y],
        [mu_2d_x - b1_x - b2_x, mu_2d_y - b1_y - b2_y]
    ]
    
    min_x = min(c[0] for c in corners)
    max_x = max(c[0] for c in corners)
    min_y = min(c[1] for c in corners)
    max_y = max(c[1] for c in corners)
    
    # Convert to integer pixel coordinates (inclusive)
    bbox = [
        int(math.floor(min_x)),
        int(math.floor(min_y)),
        int(math.ceil(max_x)),
        int(math.ceil(max_y))
    ]
    
    return {
        'mu_2d': [mu_2d_x, mu_2d_y],
        'cov2d': Sigma_2d,
        'cov2d_inv': cov2d_inv,
        'depth': z_c,
        'bbox': bbox,
        'cull': False
    }

def quaternion_to_matrix(q):
    """Converts quaternion (w,x,y,z) to 3x3 rotation matrix."""
    w, x, y, z = q
    # Normalize
    norm = math.sqrt(w*w + x*x + y*y + z*z)
    w, x, y, z = w/norm, x/norm, y/norm, z/norm
    
    xx, yy, zz = x*x, y*y, z*z
    xy, xz, yz = x*y, x*z, y*z
    wx, wy, wz = w*x, w*y, w*z
    
    return [
        [1 - 2*(yy + zz), 2*(xy - wz), 2*(xz + wy)],
        [2*(xy + wz), 1 - 2*(xx + zz), 2*(yz - wx)],
        [2*(xz - wy), 2*(yz + wx), 1 - 2*(xx + yy)]
    ]

def mat_mul(A, B):
    """Multiplies two matrices of compatible dimensions."""
    rows_a, cols_a = len(A), len(A[0])
    rows_b, cols_b = len(B), len(B[0])
    C = [[0.0]*cols_b for _ in range(rows_a)]
    for i in range(rows_a):
        for j in range(cols_b):
            for k in range(cols_a):
                C[i][j] += A[i][k] * B[k][j]
    return C

def mat_mul_vec(M, v):
    """Multiplies matrix by vector."""
    res = [0.0]*len(M)
    for i in range(len(M)):
        for j in range(len(M[0])):
            res[i] += M[i][j] * v[j]
    return res

def transpose(M):
    """Transposes a matrix."""
    rows = len(M)
    cols = len(M[0])
    T = [[0.0]*rows for _ in range(cols)]
    for i in range(rows):
        for j in range(cols):
            T[j][i] = M[i][j]
    return T
```

## Subtopics

### [Adaptive Bounding Box Computation](1206-adaptive-bounding-box-computation.md)
Computes a tight, axis-aligned bounding box (AABB) for a 2D Gaussian splat based on an opacity threshold, replacing the loose square bounding box used in original 3D Gaussian Splatting (3DGS) to determine tile intersections. It sets itself apart by deriving the box dimensions directly from the projected covariance matrix and opacity, significantly reducing false-positive splat duplications during rasterization compared to the original method.

### [Render Means](2070-render-means.md)
Render Means is the simplest possible rendering mode for 3D Gaussian Splatting, where each Gaussian is represented by a single pixel at the location where its 3D mean projects onto the 2D image plane. It demonstrates the basic camera transformation pipeline (world → camera → image) and frustum culling without any rasterization complexity. Unlike its sibling [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md), which computes screen-space covariance matrices for elliptical splatting, Render Means ignores shape and spread, treating Gaussians as point sprites.

### [Optimal Gaussian Splatting Projection](3311-optimal-gaussian-splatting-projection.md)
Optimal Gaussian Splatting Projection is a rendering method that minimizes projection error by projecting each 3D Gaussian onto a plane tangent to the line connecting the Gaussian mean to the camera center, rather than a fixed image plane. It sets itself apart from standard 3D-GS by using a unique tangent plane for each Gaussian based on its position relative to the camera, significantly reducing artifacts like blurriness and needle-like structures, particularly in wide-field-of-view or short focal length scenarios.

### [2D Gaussian Covariance Stabilization and Rescaling](3467-2d-gaussian-covariance-stabilization-and-rescaling.md)
This method stabilizes the screen-space projection of 3D Gaussians by applying a low-pass filter to the 2D covariance matrix, ensuring that each Gaussian spans at least the width of a single pixel to reduce aliasing artifacts. It computes the inverse covariance (conic) matrix and a corresponding rescale factor that preserves the integral of the Gaussian function after the covariance modification. It builds on [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md) by adding explicit anti-aliasing rescaling rather than just numerical stabilization.

### [Projection Jacobian Computation](4272-projection-jacobian-computation.md)
Computes the $2 \times 3$ Jacobian matrix of the perspective projection function with respect to 3D coordinates, used for linearizing the projection of 3D Gaussians into screen space. It transforms the 3D covariance structure into 2D screen-space covariance, enabling accurate Gaussian splatting.

### [Per-Primitive Texture Mapping for Gaussian Splatting](8622-per-primitive-texture-mapping-for-gaussian-splatting.md)
Assigns material properties as texture maps in the splat's local coordinate system rather than single attribute values per primitive, enabling higher-frequency spatial detail without increasing primitive count.

### [2D Covariance Correction via Linear Regression](9912-2d-covariance-correction-via-linear-regression.md)
A technique to approximate the relationship between projected 2D covariances (obtained by integrating along one axis using Simpson's rule) and actual 2D covariances (derived by removing the last row/column of the 3D covariance matrix) using a linear regression model instead of a neural network.

### Ellipse Parameter Extraction
Extracts the semi-major/minor axes, rotation angle, and center position from a 2D covariance matrix [S4, unofficial]. It computes semi-axes using the trace and determinant of the covariance matrix and derives rotation components using half-angle identities. A pitfall is that if the discriminant is zero, the inverse is undefined, requiring assumptions of positive-definiteness [S4, unofficial].

### Covariance Correction Factor
A scalar correction factor applied to the covariance matrix to approximate the relationship between projected 2D covariances and actual 2D covariances, replacing a more complex MLP or Simpson's rule integration [S9]. Typical values are 1.38f for inverse multiquadric correction and 1.3f for parabolic correction [S9].

### Skip Zero-Jacobian Matrix Multiplication
An optimization in the projection stage that skips multiplications involving zero elements in the Jacobian matrix, reducing computation by 52.53% compared to the original 3DGS projection step [S10]. It increases throughput from 35.94% to 75.70% (2.11× gain), raising FPS from 46.4 to 97.7 at 1080p [S10].

## Operating characteristics
**Time Complexity:** $O(M)$ for projection of $M$ Gaussians, where each projection involves constant-time matrix operations [S3, unofficial]. Sorting is $O(M \log M)$ [S2, unofficial]. Rasterization is $O(N \cdot K)$ where $N$ is pixels and $K$ is average splats per pixel [S2, unofficial].
**Memory Complexity:** $O(M)$ to store projected Gaussians [S2, unofficial].
**Performance:** Enables real-time rendering of millions of Gaussians (e.g., 134 fps on Mip-NeRF360) [S3, unofficial]. The closed-form projection avoids ray marching, which is the bottleneck in neural radiance fields [S3, unofficial].
**Strengths:** Differentiable, fast, handles view-dependent color via SH, preserves Gaussian form locally [S3, unofficial].
**Limitations:** The linearization approximation sacrifices negligible global geometric accuracy to unlock immense computational speed [S3, unofficial]. Aliasing can occur if the low-pass filter is insufficient [S5, unofficial].

## Background
Originally described by Zwicker et al. in 2002 for anti-aliased splatting (EWA Volume Splatting) [S3, unofficial]. It was reused by Kerbl et al. in 2023 for 3D Gaussian Splatting (3DGS) to enable real-time rendering of explicit 3D Gaussians [S3, unofficial]. The method sacrifices negligible global geometric accuracy to unlock immense computational speed [S3, unofficial].

## Sources
[S1] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S2] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S3] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S4] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S5] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S6] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S7] 驚くほどキレイな三次元シーン復元、「3D Gaussian Splatting」を徹底的に解説する, https://qiita.com/scomup/items/d5790da25a846e645de1, fetched 2026-10-29 (unofficial)
[S8] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
