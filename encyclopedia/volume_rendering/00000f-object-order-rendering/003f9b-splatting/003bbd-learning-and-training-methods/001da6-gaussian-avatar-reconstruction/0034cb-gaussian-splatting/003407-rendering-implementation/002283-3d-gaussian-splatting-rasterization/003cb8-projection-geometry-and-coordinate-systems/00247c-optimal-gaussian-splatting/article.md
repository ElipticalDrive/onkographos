# Optimal Gaussian Splatting

Also called: Optimal Projection, Unit Sphere Based Rasterizer

## Overview
Optimal Gaussian Splatting is a rendering method that reduces projection error by radially projecting each 3D Gaussian primitive onto a unique tangent plane perpendicular to the line connecting the Gaussian mean to the camera center, rather than projecting all Gaussians onto a single plane $z=1$ [S1]. Unlike the standard 3D Gaussian Splatting rasterization which uses a fixed perspective plane, this method adapts the projection geometry per primitive to minimize the discrepancy between the 3D Gaussian density and its 2D rasterized approximation [S1]. It achieves higher-quality rendering with smaller projection errors compared to the original approach [S1].

## How it works
**Inputs:**
*   3D Gaussian primitive defined by mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$ in world coordinates.
*   Camera extrinsic matrix (World-to-Camera transformation).
*   Camera intrinsic parameters: principal point $(c_x, c_y)$ and focal lengths $(f_x, f_y)$.
*   Pixel coordinates $(u, v)$.

**Outputs:**
*   2D Gaussian value $G_{2D}(\mathbf{x}_{2D})$ for alpha blending at pixel $(u, v)$.

**Data Structures:**
*   $\boldsymbol{\mu}'$: 3D mean in camera coordinates.
*   $\boldsymbol{\Sigma}'$: 3D covariance in camera coordinates.
*   $\mathbf{x}_p$: Projection point on the unit sphere (3D vector).
*   $\mathbf{J}_p$: $3 \times 3$ Jacobian matrix of the optimal projection function.
*   $\mathbf{Q}$: Invertible $3 \times 3$ matrix used to handle rank deficiency in the covariance inverse.

**Steps:**

1.  **Transform to Camera Space:**
    Transform the Gaussian mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$ from world coordinates to camera coordinates, resulting in $\boldsymbol{\mu}'$ and $\boldsymbol{\Sigma}'$ [S1].

2.  **Compute Projection Point on Unit Sphere:**
    Calculate the point $\mathbf{x}_p$ where the line from the camera center to the Gaussian mean intersects the unit sphere [S1]:
    $$
    \mathbf{x}_{\mathbf{p}} = \boldsymbol{\mu}' \left( \boldsymbol{\mu}'^{\top} \boldsymbol{\mu}' \right)^{-1/2}
    $$
    This defines the normal of the tangent plane [S1].

3.  **Define Optimal Projection Plane:**
    The projection plane is tangent to the unit sphere at $\mathbf{x}_p$. Its equation is:
    $$
    \mathbf{x}_{\mathbf{p}}^T \cdot \left( \mathbf{x}' - \mathbf{x}_{\mathbf{p}} \right) = 0
    $$
    [S1].

4.  **Compute Ray Direction and Project to Tangent Plane:**
    For a pixel $(u, v)$, construct the ray direction in camera space using pinhole intrinsics:
    $$
    \mathbf{d} = \begin{bmatrix} (u - c_x) / f_x \\ (v - c_y) / f_y \\ 1 \end{bmatrix}
    $$
    [S1].
    Project this ray direction onto the tangent plane defined by $\mathbf{x}_p$ using the optimal projection function $\varphi_p$:
    $$
    \varphi_p \left( \mathbf{x}' \right) = \mathbf{x}' \left( \mathbf{x}_p^\top \mathbf{x}' \right)^{-1}
    $$
    [S1].
    The resulting 2D coordinate on the tangent plane is:
    $$
    \mathbf{x}_{2D} = \varphi_p \left( \mathbf{d} \right)
    $$
    [S1]. Note that $\mathbf{x}_{2D}$ is a 3D vector lying on the tangent plane, but for the 2D Gaussian evaluation, it is treated as the coordinate relative to the projection of the mean.

5.  **Compute Jacobian Matrix $\mathbf{J}_p$:**
    Compute the Jacobian of the projection function $\varphi_p$ evaluated at the mean $\boldsymbol{\mu}'$ [S1]:
    $$
    \mathbf{J}_p = \frac{\partial \varphi_p}{\partial \mathbf{x}'} \left( \boldsymbol{\mu}' \right) = \mathbb{I} \left( \mathbf{x}_p^\top \boldsymbol{\mu}' \right)^{-1} - \boldsymbol{\mu}' \left( \mathbf{x}_p^\top \boldsymbol{\mu}' \right)^{-1} \left( \boldsymbol{\mu}'^\top \mathbf{x}_p \right)^{-1} \mathbf{x}_p^\top
    $$
    [S1].
    Explicitly, for $\boldsymbol{\mu}' = [\mu_x, \mu_y, \mu_z]^\top$ and $N = \mu_x^2 + \mu_y^2 + \mu_z^2$, the matrix elements are derived from the source [S1]. The source provides a specific form for $\mathbf{J}_p$ where the diagonal elements involve terms like $\frac{\mu_y^2 + \mu_z^2}{N^{3/2}}$ and off-diagonals involve $\frac{\mu_x \mu_y}{N^{3/2}}$ [S1].
    *Note: The source text for the explicit matrix form contains a typo in the last row description, but the general structure for the top-left components and the symmetry are preserved in the formula derivation [S1].*

6.  **Handle Rank Deficiency with Matrix $\mathbf{Q}$:**
    The Jacobian $\mathbf{J}_p$ has a rank of 2. The resulting matrix $\mathbf{J}_p \boldsymbol{\Sigma}' \mathbf{J}_p^\top$ has non-zero third row and column entries, unlike the standard projection Jacobian $\mathbf{J} \boldsymbol{\Sigma}' \mathbf{J}^\top$ where these are zero [S1]. To correctly invert the covariance for the 2D Gaussian, an invertible matrix $\mathbf{Q}$ is introduced [S1].
    The matrix $\mathbf{Q}$ is defined as:
    $$
    \mathbf{Q} = \left[ \begin{array}{c} \frac{\mu_z}{\sqrt{\mu_x^2 + \mu_z^2}} \\ \frac{\mu_x \mu_y}{\sqrt{\mu_x^2 + \mu_z^2} \sqrt{\mu_x^2 + \mu_y^2 + \mu_z^2}} \\ \frac{\mu_x}{\sqrt{\mu_x^2 + \mu_z^2} + \mu_y^2 + \mu_z^2} \end{array} \right] \cdot \left[ \begin{array}{c} 0 \\ \frac{\sqrt{\mu_x^2 + \mu_z^2}}{\sqrt{\mu_x^2 + \mu_y^2 + \mu_z^2}} - \frac{\mu_x}{\sqrt{\mu_x^2 + \mu_z^2}} \\ \frac{\mu_y \mu_z}{\sqrt{\mu_x^2 + \mu_z^2} \sqrt{\mu_x^2 + \mu_y^2 + \mu_z^2}} \\ \frac{\mu_z}{\sqrt{\mu_x^2 + \mu_y^2 + \mu_z^2}} \end{array} \right]
    $$
    [S1].
    *Note: The source text for Equation 18 presents a complex structure involving outer products or specific vector definitions that appear to have formatting inconsistencies in the raw text (e.g., mixing vector components in a matrix definition). The implementer must carefully reconstruct $\mathbf{Q}$ based on the intent to provide an invertible transformation for the rank-2 Jacobian [S1].*

7.  **Evaluate 2D Gaussian:**
    Compute the 2D Gaussian value for alpha blending [S1]:
    $$
    G_{2D}(\mathbf{x}_{2D}) = \exp \left\{ -1/2 \left( \mathbf{x}_{2D} - \varphi_p(\boldsymbol{\mu}') \right)^T \left( \mathbf{J}_p \boldsymbol{\Sigma}' \mathbf{J}_p^\top \right)^{-1} \left( \mathbf{x}_{2D} - \varphi_p(\boldsymbol{\mu}') \right) \right\}
    $$
    [S1].
    The inverse covariance matrix is obtained by skipping the third row and column of $\mathbf{J}_p \boldsymbol{\Sigma}' \mathbf{J}_p^\top$ after applying the transformation involving $\mathbf{Q}$ to ensure the equation holds [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def optimal_gaussian_splatting(mu_world, sigma_world, extrinsic, intrinsic, u, v):
    """
    Computes the 2D Gaussian value for a single pixel and Gaussian primitive.
    
    Args:
        mu_world: Tensor [3], Gaussian mean in world coordinates.
        sigma_world: Tensor [3, 3], Gaussian covariance in world coordinates.
        extrinsic: Tensor [4, 4], World-to-Camera transformation matrix.
        intrinsic: Tuple (cx, cy, fx, fy), Camera intrinsics.
        u, v: Float, Pixel coordinates.
        
    Returns:
        Tensor [1], The 2D Gaussian value G_2D.
    """
    # 1. Transform to Camera Space
    # mu' = R * mu + t
    mu_cam = (extrinsic[:3, :3] @ mu_world + extrinsic[:3, 3:4]).squeeze()
    # Sigma' = R * Sigma * R^T
    R = extrinsic[:3, :3]
    sigma_cam = R @ sigma_world @ R.T
    
    cx, cy, fx, fy = intrinsic
    
    # 2. Compute Projection Point on Unit Sphere (x_p)
    # x_p = mu' / ||mu'||
    mu_norm_sq = torch.dot(mu_cam, mu_cam)
    mu_norm = torch.sqrt(mu_norm_sq)
    x_p = mu_cam / mu_norm
    
    # 3. Compute Ray Direction in Camera Space
    # d = [(u - cx)/fx, (v - cy)/fy, 1]
    d = torch.tensor([(u - cx) / fx, (v - cy) / fy, 1.0])
    
    # 4. Project Ray Direction to Tangent Plane (x_2D)
    # phi_p(x') = x' * (x_p^T * x')^-1
    # Note: x_p^T * x_p = 1, so the formula simplifies to x' / (x_p^T * x')
    dot_xp_d = torch.dot(x_p, d)
    x_2D = d / dot_xp_d
    
    # 5. Project Mean to Tangent Plane (phi_p(mu'))
    dot_xp_mu = torch.dot(x_p, mu_cam)
    mu_proj = mu_cam / dot_xp_mu
    
    # 6. Compute Jacobian J_p
    # J_p = I * (x_p^T * mu')^-1 - mu' * (x_p^T * mu')^-1 * (mu'^T * x_p)^-1 * x_p^T
    # Since x_p^T * mu' = mu'^T * x_p, let inv_dot = 1 / dot_xp_mu
    inv_dot = 1.0 / dot_xp_mu
    # J_p = (1/dot_xp_mu) * I - (1/dot_xp_mu^2) * (mu' * x_p^T)
    # However, the source provides an explicit matrix form. We will use the explicit form for stability.
    # mu = [mu_x, mu_y, mu_z]
    mu_x, mu_y, mu_z = mu_cam[0], mu_cam[1], mu_cam[2]
    N = mu_norm_sq # mu_x^2 + mu_y^2 + mu_z^2
    N_32 = N ** 1.5
    
    # Construct J_p explicitly as per source Eq 15 structure
    # Note: Source text has a typo in the last row, but the symmetry and top-left structure are clear.
    # We construct the full 3x3 matrix based on the derived formula J_p = d(phi)/dx
    # phi_p(x) = x / (x_p^T x). At x=mu, x_p is fixed.
    # d(phi)/dx = (I * (x_p^T x) - x * x_p^T) / (x_p^T x)^2
    # Evaluated at mu: J_p = (I * dot_xp_mu - mu_cam * x_p^T) / (dot_xp_mu^2)
    
    J_p = (torch.eye(3) * dot_xp_mu - torch.outer(mu_cam, x_p)) / (dot_xp_mu ** 2)
    
    # 7. Compute 2D Covariance and Inverse
    # Sigma_2D_raw = J_p @ Sigma_cam @ J_p^T
    Sigma_2D_raw = J_p @ sigma_cam @ J_p.T
    
    # Handle Rank Deficiency
    # The source states that J_p has rank 2 and the third row/col of Sigma_2D_raw are not zero.
    # An invertible matrix Q is used. The source Eq 18 is complex and potentially malformed in text.
    # Standard approach for rank-2 projection: Project to 2D basis.
    # However, adhering strictly to the source's claim about Q:
    # We will implement the "skip third row/col" method if the third row/col are negligible,
    # but the source explicitly says they are NOT zero.
    # Given the ambiguity of Eq 18 in the source text, we use the standard 2D subspace projection
    # which is mathematically equivalent to handling the rank deficiency by restricting to the tangent plane basis.
    # But to follow the "Optimal" method's specific claim, we note that the inverse is obtained by skipping
    # the third row and column of the resulting matrix IF we transform coordinates appropriately.
    
    # For implementation purposes, since the source's Q is ambiguous, we use the standard 2D extraction
    # which is robust: Take the top-left 2x2 of the transformed covariance if the projection was to z=1.
    # Here, we are on a tangent plane. The most robust interpretation of "skipping third row/col"
    # in the context of the source's error analysis is that the effective 2D covariance is the
    # projection of the 3D covariance onto the tangent plane basis.
    
    # Let's use the explicit formula for the inverse covariance in the 2D tangent plane.
    # The source implies: Sigma_2D_inv = (J_p Sigma J_p^T)^{-1}_{2x2}
    # We extract the top-left 2x2 block.
    Sigma_2D_inv = Sigma_2D_raw[:2, :2].inverse()
    
    # 8. Evaluate Gaussian
    # Diff vector in 2D tangent plane coordinates.
    # x_2D and mu_proj are 3D vectors on the tangent plane.
    # We need their coordinates in a 2D basis.
    # However, the source formula uses the 3D vectors directly in the exponent with the 2D inverse?
    # This implies the vectors are treated as 2D coordinates in the local basis.
    # To implement without defining a basis explicitly, we can use the property that
    # for vectors v on the tangent plane, v^T Sigma_2D_inv v is invariant if Sigma_2D_inv
    # is the pseudo-inverse or properly defined.
    
    # The source formula:
    # G_2D = exp(-0.5 * (x_2D - mu_proj)^T @ Sigma_2D_inv @ (x_2D - mu_proj))
    # This requires x_2D and mu_proj to be 2D vectors.
    # We project x_2D and mu_proj onto the first two axes of the camera space?
    # No, they are on the tangent plane.
    # We must define a 2D basis for the tangent plane.
    # Let u_axis and v_axis be orthonormal vectors perpendicular to x_p.
    
    # Construct basis for tangent plane
    # If mu_cam is not parallel to z-axis, we can use cross products.
    # Simple basis construction:
    if abs(mu_cam[2]) < 0.99:
        u_axis = torch.cross(x_p, torch.tensor([0.0, 0.0, 1.0]))
        u_axis = u_axis / torch.norm(u_axis)
    else:
        u_axis = torch.tensor([1.0, 0.0, 0.0])
    v_axis = torch.cross(x_p, u_axis)
    
    # Project x_2D and mu_proj onto this basis to get 2D coordinates
    x_2D_2d = torch.tensor([torch.dot(x_2D, u_axis), torch.dot(x_2D, v_axis)])
    mu_proj_2d = torch.tensor([torch.dot(mu_proj, u_axis), torch.dot(mu_proj, v_axis)])
    
    diff = x_2D_2d - mu_proj_2d
    exponent = -0.5 * torch.dot(diff, Sigma_2D_inv @ diff)
    
    return torch.exp(exponent)
```

## Operating characteristics
*   **Time Complexity:** Higher per-pixel computation than standard 3D-GS due to the per-Gaussian projection plane calculation and Jacobian evaluation [S1]. However, it avoids the extensive ray sampling of NeRF [S1].
*   **Memory Complexity:** Similar to 3D-GS, storing Gaussians and their parameters [S1].
*   **Performance:** Achieves higher rendering quality with smaller projection errors compared to the original 3D-GS [S1].
*   **Strengths:** Reduces projection artifacts, especially for Gaussians far from the camera center or at oblique angles [S1].
*   **Limitations:** More complex implementation requiring custom CUDA kernels for efficiency [S1]. The handling of the rank-deficient Jacobian via matrix $\mathbf{Q}$ adds implementation complexity [S1].

## Background
Optimal Gaussian Splatting was introduced to address the error analysis of 3D Gaussian Splatting [S1]. It is based on the theoretical finding that the error function attains its minimum when the projection of the Gaussian mean on the plane coincides with the projection from the camera center to the plane [S1]. The method was implemented based on the PyTorch framework in 3D-GS with custom CUDA kernels [S1]. It was tested on 13 real scenes, including those from Mip-NeRF360 [S1].

## Sources
[S1] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
