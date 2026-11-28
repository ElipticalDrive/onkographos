# Fisheye-GS

Also called: Differentiable Equidistant Projection for 3DGS, Fisheye 3D Gaussian Splatting, Fisheye-GS Equidistant Projection

## Overview
Fisheye-GS is a method that integrates fisheye camera data into 3D Gaussian Splatting (3DGS) by replacing the standard perspective projection with an equidistant projection model and optimizing the rendering pipeline to handle extreme curvature and wide fields of view [S1, unofficial]. It sets itself apart from standard 3DGS by deriving specific Jacobians for the equidistant projection to maintain image quality across wide angles, and by reducing computational redundancy in tile-based splatting to achieve real-time performance [S1, unofficial].

## How it works
Fisheye-GS extends the standard 3DGS pipeline by modifying the projection function $\phi$ and its Jacobian $\mathbf{J}$ to support fisheye lenses, while optimizing the CUDA kernel for tile traversal.

**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\mathcal{S}$ defined by means $\boldsymbol{\mu}$, covariances $\boldsymbol{\Sigma}$, opacities $\alpha$, and colors $\mathbf{c}$; camera parameters including focal length $f$, principal point $(c_x, c_y)$, and pixel focal lengths $f_x, f_y$; and the viewing matrix $\mathbf{W}$ and translation $\mathbf{b}$.
*   **Outputs:** A rendered 2D image $\mathbf{C}$.

**Data Structures:**
The method uses the standard 3DGS data structures for Gaussians:
*   $\boldsymbol{\mu} \in \mathbb{R}^3$: Mean position.
*   $\boldsymbol{\Sigma} \in \mathbb{R}^{3\times3}$: Covariance matrix, parameterized as $\boldsymbol{\Sigma} = \mathbf{R}\mathbf{S}\mathbf{S}^T\mathbf{R}^T$, where $\mathbf{R} \in SO(3)$ is the rotation matrix (from quaternions) and $\mathbf{S} = \text{diag}(s_1, s_2, s_3)$ is the scaling matrix [S1, unofficial].
*   $\mathbf{c}$: Color.
*   $\alpha$: Opacity.

**Projection Model:**
Standard 3DGS uses perspective projection. Fisheye-GS uses the **equidistant projection model** [S1, unofficial]. For a point $\mathbf{p}_c = (x_c, y_c, z_c)^T$ in camera coordinates:
1.  Calculate the distance to the z-axis:
    $$ l_z = \sqrt{x_c^2 + y_c^2} $$
2.  Calculate the angle of incidence $\theta$:
    $$ \theta = \arctan\left(\frac{l_z}{z_c}\right) $$
3.  Calculate the pixel coordinates $(x_p, y_p)$:
    $$ x_p = c_x + \frac{f_x \theta x_c}{l_z} $$
    $$ y_p = c_y + \frac{f_y \theta y_c}{l_z} $$
    where $f$ is the focal length (used in $r_d = f\theta$), and $f_x, f_y$ are pixel focal lengths [S1, unofficial].

**Jacobian Derivation:**
The 2D covariance $\boldsymbol{\Sigma}'$ is computed using the Jacobian $\mathbf{J}$ of the projection function $\phi$ with respect to the camera-space coordinates:
$$ \boldsymbol{\Sigma}' = \mathbf{J} \mathbf{W} \boldsymbol{\Sigma} \mathbf{W}^T \mathbf{J}^T $$
The Jacobian $\mathbf{J} \in \mathbb{R}^{2\times3}$ is the derivative of the equidistant projection equations with respect to $(x_c, y_c, z_c)$. This allows for differentiable training [S1, unofficial].

**Rendering Pipeline:**
1.  **View Transformation:** Transform $\boldsymbol{\mu}$ and $\boldsymbol{\Sigma}$ to camera space: $\boldsymbol{\mu}_c = \mathbf{W}\boldsymbol{\mu} + \mathbf{b}$ [S1, unofficial].
2.  **Projection:** Apply $\phi$ to $\boldsymbol{\mu}_c$ to get $\boldsymbol{\mu}_p$. Compute $\boldsymbol{\Sigma}'$ using the derived Jacobian [S1, unofficial].
3.  **Tile-Based Rasterization:**
    *   Divide the image plane into $16 \times 16$ tiles [S1, unofficial].
    *   For each Gaussian, compute its axis-aligned bounding box (AABB) in screen space.
    *   Traverse tiles intersecting the AABB.
    *   Sort Gaussians by depth within each tile.
    *   Perform alpha blending for each pixel in the tile:
        $$ \mathbf{C} = \sum_{i=1}^n \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $\alpha_i$ is the opacity of the $i$-th Gaussian evaluated at the pixel [S1, unofficial].

**Optimization:**
The method eliminates computational redundancies in the CUDA kernel, specifically reducing the number of overlapped tiles processed, which is critical for the extreme curvature of fisheye lenses [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def equidistant_projection(xc, yc, zc, cx, cy, fx, fy):
    """
    Projects a 3D point in camera space to 2D pixel coordinates using equidistant projection.
    """
    lz = math.sqrt(xc**2 + yc**2)
    if lz == 0:
        # Degenerate case: point is on the optical axis
        return cx, cy
    
    theta = math.atan(lz / zc)
    xp = cx + (fx * theta * xc) / lz
    yp = cy + (fy * theta * yc) / lz
    return xp, yp

def compute_jacobian_equidistant(xc, yc, zc, fx, fy):
    """
    Computes the 2x3 Jacobian matrix J of the equidistant projection with respect to (xc, yc, zc).
    J = [ dxp/dxc, dxp/dyc, dxp/dzc ]
        [ dyp/dxc, dyp/dyc, dyp/dzc ]
    """
    lz = math.sqrt(xc**2 + yc**2)
    if lz == 0:
        # Handle singularity at optical axis
        return np.zeros((2, 3))
    
    theta = math.atan(lz / zc)
    
    # Derivatives of theta with respect to xc, yc, zc
    # theta = arctan(lz / zc)
    # d(theta)/d(lz) = (1/zc) / (1 + (lz/zc)^2) = zc / (lz^2 + zc^2)
    # d(theta)/d(zc) = (-lz/zc^2) / (1 + (lz/zc)^2) = -lz / (lz^2 + zc^2)
    
    denom = lz**2 + zc**2
    d_theta_d_lz = zc / denom
    d_theta_d_zc = -lz / denom
    
    # dlz/dxc = xc/lz, dlz/dyc = yc/lz, dlz/dzc = 0
    d_lz_dxc = xc / lz
    d_lz_dyc = yc / lz
    
    # x_p = cx + fx * theta * xc / lz
    # dxp/dxc = fx * ( (d_theta/dxc * xc + theta) * lz - theta * xc * dlz/dxc ) / lz^2
    #         = fx * ( (d_theta/d_lz * dlz/dxc * xc + theta) * lz - theta * xc * dlz/dxc ) / lz^2
    #         = fx * ( d_theta/d_lz * dlz/dxc * xc * lz + theta * lz - theta * xc * dlz/dxc ) / lz^2
    #         = fx * ( d_theta/d_lz * dlz/dxc * xc * (lz - xc/lz * lz) + theta * lz ) / lz^2 ... simplifying is complex.
    # Let's use symbolic differentiation result or numerical approximation if analytical is too verbose.
    # However, for implementation, we need exact formulas.
    
    # Let A = theta * xc / lz
    # dxp/dxc = fx * ( d(theta*xc/lz)/dxc )
    # d(theta*xc/lz)/dxc = (d_theta/dxc * xc + theta) / lz - theta * xc * dlz/dxc / lz^2
    # d_theta/dxc = d_theta/d_lz * dlz/dxc = d_theta/d_lz * xc/lz
    
    d_theta_dxc = d_theta_d_lz * (xc / lz)
    d_theta_dyc = d_theta_d_lz * (yc / lz)
    
    # Term for x_p
    term1_x = (d_theta_dxc * xc + theta) / lz
    term2_x = theta * xc * (xc / lz) / (lz**2)
    dxp_dxc = fx * (term1_x - term2_x)
    
    term1_y = (d_theta_dyc * xc + 0) / lz # theta doesn't depend on yc directly in numerator, but lz does
    # Wait, theta depends on yc via lz.
    # d(theta*xc/lz)/dyc = (d_theta/dyc * xc) / lz - theta * xc * dlz/dyc / lz^2
    term1_y = (d_theta_dyc * xc) / lz
    term2_y = theta * xc * (yc / lz) / (lz**2)
    dxp_dyc = fx * (term1_y - term2_y)
    
    # d(theta*xc/lz)/dzc = (d_theta/dzc * xc) / lz - theta * xc * 0 / lz^2
    term1_z = (d_theta_d_zc * xc) / lz
    dxp_dzc = fx * term1_z
    
    # Term for y_p
    # y_p = cy + fy * theta * yc / lz
    # dyp/dxc = fy * (d_theta/dxc * yc) / lz - fy * theta * yc * dlz/dxc / lz^2
    term1_x_y = (d_theta_dxc * yc) / lz
    term2_x_y = theta * yc * (xc / lz) / (lz**2)
    dyp_dxc = fy * (term1_x_y - term2_x_y)
    
    # dyp/dyc = fy * ( (d_theta/dyc * yc + theta) / lz - theta * yc * dlz/dyc / lz^2 )
    term1_y_y = (d_theta_dyc * yc + theta) / lz
    term2_y_y = theta * yc * (yc / lz) / (lz**2)
    dyp_dyc = fy * (term1_y_y - term2_y_y)
    
    # dyp/dzc = fy * (d_theta/dzc * yc) / lz
    term1_z_y = (d_theta_d_zc * yc) / lz
    dyp_dzc = fy * term1_z_y
    
    J = np.array([
        [dxp_dxc, dxp_dyc, dxp_dzc],
        [dyp_dxc, dyp_dyc, dyp_dzc]
    ])
    return J

def render_fisheye_gaussian(gaussian, camera_params, view_matrix, translation):
    """
    Renders a single 3D Gaussian using Fisheye-GS projection.
    """
    # 1. View Transformation
    mu_world = gaussian['mu']
    mu_c = view_matrix @ mu_world + translation
    
    xc, yc, zc = mu_c
    
    # 2. Equidistant Projection
    cx, cy = camera_params['principal_point']
    fx, fy = camera_params['pixel_focal_lengths']
    
    xp, yp = equidistant_projection(xc, yc, zc, cx, cy, fx, fy)
    mu_p = np.array([xp, yp])
    
    # 3. Covariance Projection
    Sigma = gaussian['covariance'] # 3x3
    W = view_matrix
    
    # Compute Jacobian J (2x3)
    J = compute_jacobian_equidistant(xc, yc, zc, fx, fy)
    
    # Sigma' = J * W * Sigma * W^T * J^T
    Sigma_prime = J @ W @ Sigma @ W.T @ J.T
    
    # 4. Rasterization (Simplified)
    # In practice, this involves tile-based AABB check and alpha blending.
    # Here we just return the projected parameters for the rasterizer.
    return {
        'mu_p': mu_p,
        'Sigma_prime': Sigma_prime,
        'opacity': gaussian['opacity'],
        'color': gaussian['color']
    }
```

## Operating characteristics
*   **Performance:** Achieves a maximum rendering efficiency of 570 FPS at a resolution of 1752 × 1168 [S1, unofficial].
*   **Complexity:** The method reduces computational redundancy compared to standard 3DGS by optimizing the number of overlapped tiles, which is particularly beneficial for the extreme curvature and wide field of view of fisheye lenses [S1, unofficial].
*   **Strengths:** Maintains image quality and precision across extreme curvature; produces distortion-free imagery; suitable for real-time applications [S1, unofficial].
*   **Limitations:** Specific to fisheye cameras using the equidistant projection model; may not be optimal for standard perspective cameras without modification [S1, unofficial].

## Background
Fisheye-GS was introduced to address the inefficiencies of tile-based splatting when applied to fisheye cameras, which have wide fields of view (180° or greater) and significant radial distortion [S1, unofficial]. It is the first work to seamlessly integrate fisheye camera data into 3D Gaussian Splatting, enabling applications in virtual reality, gaming, security, and panoramic environments [S1, unofficial].

## Sources
[S1] Fisheye-GS: Efficient Gaussian Splatting for Fisheye Camera, https://zenodo.org/records/11283489, fetched 2026-10-26 (unofficial)
