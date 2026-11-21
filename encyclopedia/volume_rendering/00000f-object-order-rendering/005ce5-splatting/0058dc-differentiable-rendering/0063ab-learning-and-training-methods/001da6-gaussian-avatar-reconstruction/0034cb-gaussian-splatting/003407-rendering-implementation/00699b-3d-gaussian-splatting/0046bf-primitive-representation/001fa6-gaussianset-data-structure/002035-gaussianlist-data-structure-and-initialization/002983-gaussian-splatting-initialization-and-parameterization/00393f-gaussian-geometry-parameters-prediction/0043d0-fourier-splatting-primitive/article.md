# Fourier Splatting Primitive

Also called: Fourier encoded primitive, Fourier surfel

## Overview
The Fourier Splatting primitive is a planar surfel for radiance field rendering that replaces the fixed Gaussian or elliptical falloff with a boundary defined by a Fourier series, allowing the representation of arbitrary closed shapes [S1]. Unlike 3D Gaussian Splatting (3DGS) or 2D Gaussian Splatting (2DGS), which are limited to fixed elliptical or disc shapes, this primitive enables higher fidelity representation of complex geometry and fine-grained details [S1]. Its key differentiator is inherent scalability: the level of detail (LoD) is controlled by truncating Fourier coefficients at runtime, allowing a single trained model to be rendered at varying complexities without retraining or post-hoc compression [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Camera parameters (view/projection matrices), a set of primitives $\{s_i\}$, and a target resolution [S1].
*   **Output:** A rendered image with color and opacity per pixel [G].

**Data Structures**
Each primitive $s_i$ is parameterized by the following fields [S1]:
1.  **Center:** $p_i \in \mathbb{R}^3$.
2.  **Rotation:** $q_i \in \mathbb{R}^4$ (unit quaternion) defining the local tangent frame $(t_u, t_v, t_w)$ [S1].
3.  **Opacity:** $o_i \in (0, 1)$ [S1].
4.  **Appearance:** $c_i \in \mathbb{R}^{48}$ (Spherical Harmonics coefficients, degree 3) [S1].
5.  **Circumradius:** $R_i \in \mathbb{R}_{>0}$, bounding the primitive extent [S1].
6.  **Fourier Amplitudes:** $\{r_{i,k}\}_{k=0}^{K-1} \in \mathbb{R}^K$ [S1].
7.  **Fourier Phases:** $\{\phi_{i,k}\}_{k=0}^{K-1} \in \mathbb{R}$ [S1].
8.  **Sharpness:** $\sigma_i \in \mathbb{R}_{>0}$ [S1].

**Parameters**
*   $K$: Number of Fourier frequencies. Typical value: 6 [S1].
*   $K_{\text{active}}$: Number of active frequency components at the start of optimization. Typical value: 1 [S1].

**Formulas**
The boundary of the primitive in its local tangent plane is a closed curve $r_i(\theta)$ [S1]. To ensure the boundary does not exceed the circumradius $R_i$, the raw amplitudes $r_{i,k}$ are normalized using squared-$\ell_1$ normalization [S1]:

$$
\bar{r}_{i,k} = \frac{r_{i,k}^2}{\sum_{k'=0}^{K-1} r_{i,k'}^2} \quad \text{such that} \quad \sum_{k=0}^{K-1} \bar{r}_{i,k} = 1
$$

This guarantees $r_i(\theta) \le R_i$ for all $\theta$ [S1].

The boundary radius $r_i(\theta)$ is computed efficiently using Horner’s method to avoid transcendental calls for $e^{ik\theta}$ [S1]. Let $w = e^{i\theta} = (\cos \theta, \sin \theta)$ [S1]. The complex coefficients $c_{i,k}$ are derived from the normalized amplitudes and phases as $c_{i,k} = \bar{r}_{i,k} e^{i \phi_{i,k}}$ [G]. The recurrence is computed from $k = K-1$ down to $0$ [S1]:

$$
z_{K-1} = c_{i,K-1}
$$
$$
z_{k} = z_{k+1} \cdot w + c_{i,k} \quad \text{for } k = K-2, \dots, 0
$$

The final radius is:
$$
r_i(\theta) = R_i |z_0|
$$

This requires $K-1$ complex multiply-adds [S1].

**Numbered Steps**
1.  **Ray-Plane Intersection:** For each pixel, compute the intersection with the primitive's plane defined by $p_i$ and normal $t_w$ [S1]. Transform the intersection point into the local tangent frame $(t_u, t_v)$ to obtain coordinates $(u_j, v_j)$ [S1].
2.  **Polar Conversion:** Compute the polar coordinates $(\rho_j, \theta_j)$ in the tangent plane [S1].
    *   $\rho_j = \sqrt{u_j^2 + v_j^2}$
    *   $\cos \theta_j = u_j / \rho_j$
    *   $\sin \theta_j = v_j / \rho_j$
    *   [G] Handle $\rho_j \approx 0$ by setting $\theta_j = 0$ to avoid division by zero.
3.  **Boundary Evaluation:** Compute $r_i(\theta_j)$ using the Horner’s method recurrence defined above [S1].
4.  **Signed Distance:** Compute the signed distance $\varphi_j$ from the pixel to the boundary [S1]:
    $$
    \varphi_j = \rho_j - r_i(\theta_j)
    $$
5.  **Alpha Compositing:** Use a power window function (adapted from Triangle Splatting) based on $\varphi_j$ and the sharpness parameter $\sigma_i$ to determine the alpha value for the primitive at this pixel [S1].
6.  **Color Evaluation:** Evaluate the spherical harmonics using the view direction to get the color contribution [S1].
7.  **Rendering:** Composite the primitives tile-by-tile using standard alpha blending [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def quaternion_to_rotation_matrix(q):
    """Converts a unit quaternion (x, y, z, w) to a 3x3 rotation matrix [G]."""
    x, y, z, w = q
    return [
        [1 - 2*y*y - 2*z*z, 2*x*y - 2*z*w, 2*x*z + 2*y*w],
        [2*x*y + 2*z*w, 1 - 2*x*x - 2*z*z, 2*y*z - 2*x*w],
        [2*x*z - 2*y*w, 2*y*z + 2*x*w, 1 - 2*x*x - 2*y*y]
    ]

def evaluate_sh(c_sh, view_dir):
    """Evaluates degree-3 Spherical Harmonics (48 coefficients) for a view direction [G]."""
    x, y, z = view_dir
    xx, yy, zz = x*x, y*y, z*z
    xy, yz, xz = x*y, y*z, x*z
    x2 = xx - yy
    y2 = yy - zz
    z2 = 3*zz - 1
    r = 0.2820948*c_sh[0] + 0.4886025*(c_sh[1]*y + c_sh[2]*z + c_sh[3]*x) + \
        1.0925484*(c_sh[4]*yz + c_sh[5]*xz + c_sh[6]*x2 + c_sh[7]*xy + c_sh[8]*0.5*z2)
    g = 0.2820948*c_sh[9] + 0.4886025*(c_sh[10]*y + c_sh[11]*z + c_sh[12]*x) + \
        1.0925484*(c_sh[13]*yz + c_sh[14]*xz + c_sh[15]*x2 + c_sh[16]*xy + c_sh[17]*0.5*z2)
    b = 0.2820948*c_sh[18] + 0.4886025*(c_sh[19]*y + c_sh[20]*z + c_sh[21]*x) + \
        1.0925484*(c_sh[22]*yz + c_sh[23]*xz + c_sh[24]*x2 + c_sh[25]*xy + c_sh[26]*0.5*z2)
    return (r, g, b)

def compute_primitive_alpha_and_color(p_i, q_i, o_i, c_sh, R_i, r_raw, phi_raw, sigma_i, K, 
                                      ray_origin, ray_dir, img_width, img_height):
    """
    Evaluates a single Fourier Splatting primitive over a pixel grid.
    Returns alpha and color contribution arrays.
    """
    # Extract local frame [G]
    R_mat = quaternion_to_rotation_matrix(q_i)
    t_u = R_mat[0]
    t_v = R_mat[1]
    t_w = R_mat[2]
    
    alpha_grid = [0.0] * (img_height * img_width)
    color_grid = [(0.0, 0.0, 0.0)] * (img_height * img_width)
    
    # Precompute normalization [S1]
    sum_sq = sum(r**2 for r in r_raw)
    if sum_sq == 0:
        return alpha_grid, color_grid
    r_norm = [r**2 / sum_sq for r in r_raw]
    
    # Precompute complex coefficients c_{i,k} = r_norm[k] * exp(i * phi_raw[k]) [G]
    c_real = [r_norm[k] * math.cos(phi_raw[k]) for k in range(K)]
    c_imag = [r_norm[k] * math.sin(phi_raw[k]) for k in range(K)]
    
    for y in range(img_height):
        for x in range(img_width):
            # 1. Ray-Plane Intersection [S1]
            denom = t_w[0]*ray_dir[0] + t_w[1]*ray_dir[1] + t_w[2]*ray_dir[2]
            if abs(denom) < 1e-6:
                continue
            t = ((p_i[0] - ray_origin[0])*t_w[0] + (p_i[1] - ray_origin[1])*t_w[1] + (p_i[2] - ray_origin[2])*t_w[2]) / denom
            if t < 0:
                continue
            ix = ray_origin[0] + t * ray_dir[0]
            iy = ray_origin[1] + t * ray_dir[1]
            iz = ray_origin[2] + t * ray_dir[2]
            
            # 2. Local Tangent Coordinates [S1]
            vec_x = ix - p_i[0]
            vec_y = iy - p_i[1]
            vec_z = iz - p_i[2]
            u_j = vec_x*t_u[0] + vec_y*t_u[1] + vec_z*t_u[2]
            v_j = vec_x*t_v[0] + vec_y*t_v[1] + vec_z*t_v[2]
            
            # 3. Polar Conversion [S1]
            rho_j = math.sqrt(u_j*u_j + v_j*v_j)
            if rho_j < 1e-6:
                theta_j = 0.0
                cos_theta = 1.0
                sin_theta = 0.0
            else:
                theta_j = math.atan2(v_j, u_j)
                cos_theta = u_j / rho_j
                sin_theta = v_j / rho_j
                
            # 4. Horner's Method for Boundary [S1]
            z_real = c_real[K-1]
            z_imag = c_imag[K-1]
            for k in range(K-2, -1, -1):
                new_real = z_real * cos_theta - z_imag * sin_theta
                new_imag = z_real * sin_theta + z_imag * cos_theta
                z_real = new_real + c_real[k]
                z_imag = new_imag + c_imag[k]
            r_boundary = R_i * math.sqrt(z_real*z_real + z_imag*z_imag)
            
            # 5. Signed Distance [S1]
            signed_dist = rho_j - r_boundary
            
            # 6. Alpha Calculation (Power Window) [G]
            if signed_dist > 0:
                alpha = 0.0
            else:
                alpha = max(0.0, 1.0 - (signed_dist / sigma_i)**2)**2
                
            if alpha < 1e-4:
                continue
                
            # 7. Color Evaluation [S1]
            view_dir_local = (
                t_u[0]*ray_dir[0] + t_u[1]*ray_dir[1] + t_u[2]*ray_dir[2],
                t_v[0]*ray_dir[0] + t_v[1]*ray_dir[1] + t_v[2]*ray_dir[2],
                t_w[0]*ray_dir[0] + t_w[1]*ray_dir[1] + t_w[2]*ray_dir[2]
            )
            color = evaluate_sh(c_sh, view_dir_local)
            
            idx = y * img_width + x
            alpha_grid[idx] += alpha * o_i
            color_grid[idx] = (
                color_grid[idx][0] + color[0] * alpha * o_i,
                color_grid[idx][1] + color[1] * alpha * o_i,
                color_grid[idx][2] + color[2] * alpha * o_i
            )
            
    return alpha_grid, color_grid
```

**Edge Cases and Pitfalls**
*   **Ray-Plane Intersection:** For each pixel, compute the intersection with the primitive's plane defined by $p_i$ and normal $t_w$ [S1].
*   **Division by Zero:** When computing $\theta_j$, if $\rho_j \approx 0$, $\cos \theta_j$ and $\sin \theta_j$ are undefined [G].
*   **Overshooting:** Without squared-$\ell_1$ normalization, amplitudes can sum to exceed $R_i$, causing the boundary to overshoot the circumscribed circle [S1].
*   **Optimization Stability:** The optimization process begins with $K_{\text{active}} = 1$ to establish a stable global structure [S1].

## Operating characteristics
*   **Time Complexity:** The boundary evaluation uses Horner’s method, requiring $K-1$ complex multiply-adds, which is $O(K)$ per pixel per primitive [S1]. With $K=6$, this is constant and fast [S1].
*   **Memory Complexity:** Each primitive stores $K$ amplitudes and $K$ phases [S1]. With $K=6$ and degree 3 SH (48 coeffs), plus center, quaternion, opacity, and radius, the parameter count is 70 per primitive [S1].
*   **Scalability:** The method is inherently scalable. Rendering at lower LoD is achieved by truncating Fourier coefficients at runtime (e.g., using only $k < K_{\text{active}}$) without retraining [S1].
*   **Strengths:** Arbitrary shape representation allows higher fidelity for complex geometry compared to 2D/3D Gaussians [S1]. Continuous LoD control [S1].
*   **Limitations:** Requires careful initialization ($K_{\text{active}}=1$) to avoid optimization instability [S1].

## Background
The Fourier Splatting primitive was introduced in "Fourier Splatting: Generalized Fourier encoded primitives for scalable radiance fields" [S1]. It addresses the limitation of existing methods where visual fidelity is tied strictly to the number of primitives, requiring pruning for quality downscaling [S1]. It builds on the concept of planar surfels (like 2DGS) but enhances expressiveness with Fourier-encoded boundaries [S1].

## Sources
[S1] Fourier Splatting: Generalized Fourier encoded primitives for scalable radiance fields, https://doi.org/10.48550/arXiv.2603.19834, fetched 2026-10-30
