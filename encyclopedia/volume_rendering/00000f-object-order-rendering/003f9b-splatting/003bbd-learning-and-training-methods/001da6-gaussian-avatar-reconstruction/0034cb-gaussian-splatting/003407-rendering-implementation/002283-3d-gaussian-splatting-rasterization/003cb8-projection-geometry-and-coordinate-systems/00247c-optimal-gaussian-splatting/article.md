# Optimal Gaussian Splatting

Also called: Optimal Projection, Unit Sphere Based Rasterizer

## Overview
Optimal Gaussian Splatting is a rendering method that reduces projection error by radially projecting each 3D Gaussian onto a unique tangent plane perpendicular to the line connecting the Gaussian mean to the camera center, rather than projecting all Gaussians onto a single plane $z=1$ [S1]. Unlike the original 3D Gaussian Splatting rasterization which uses a fixed perspective plane, this method treats the projection as a radial mapping onto a unit sphere, ensuring the projection of the Gaussian mean coincides with the ray direction from the camera center [S1]. It differs from NeRF by avoiding extensive ray sampling, instead determining which tangent-plane Gaussians intersect the ray on the unit sphere [S1].

## How it works
**Inputs:**
*   3D Gaussian primitives defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and color $\mathbf{c}$ in world coordinates [S1].
*   Camera extrinsics (world-to-camera transformation) [S1].
*   Camera intrinsics: principal point $(c_x, c_y)$ and focal lengths $(f_x, f_y)$ [S1].
*   Image pixel coordinates $(u, v)$ [S1].

**Outputs:**
*   2D projected mean $\mathbf{x}_{2D}$ and 2D covariance $\boldsymbol{\Sigma}_{2D}$ for alpha blending [S1].

**Data Structures:**
*   Camera space mean $\boldsymbol{\mu}'$ [S1].
*   Unit sphere projection point $\mathbf{x}_p$ [S1].
*   Jacobian matrix $\mathbf{J}_p$ [S1].
*   Invertible matrix $\mathbf{Q}$ for handling rank deficiency [S1].

**Steps:**

1.  **Transform to Camera Space:**
    Transform the world coordinates of the Gaussian mean into the camera coordinate system [S1].
    $$ \boldsymbol{\mu}' = \mathbf{W} \boldsymbol{\mu} $$
    where $\mathbf{W}$ is the world-to-camera transformation matrix [S1].

2.  **Compute Projection Point on Unit Sphere:**
    Calculate the point $\mathbf{x}_p$ on the unit sphere corresponding to the Gaussian mean $\boldsymbol{\mu}'$ [S1]. This point serves as the normal to the tangent plane:
    $$ \mathbf{x}_{\mathbf{p}} = \boldsymbol{\mu}' \left( \boldsymbol{\mu}'^{\top} \boldsymbol{\mu}' \right)^{-1/2} $$
    Let $N = \boldsymbol{\mu}'^{\top} \boldsymbol{\mu}' = \mu_x'^2 + \mu_y'^2 + \mu_z'^2$. Then:
    $$ \mathbf{x}_{\mathbf{p}} = \frac{1}{\sqrt{N}} \begin{bmatrix} \mu_x' \\ \mu_y' \\ \mu_z' \end{bmatrix} $$
    [S1]

3.  **Define Optimal Projection Plane:**
    The optimal projection plane is tangent to the unit sphere at $\mathbf{x}_p$ [S1]. Its equation is:
    $$ \mathbf{x}_{\mathbf{p}}^T \cdot \left( \mathbf{x}' - \mathbf{x}_{\mathbf{p}} \right) = 0 $$
    [S1]

4.  **Compute Optimal Projection Function:**
    The projection function $\varphi_p$ maps a 3D point $\mathbf{x}'$ onto the tangent plane at $\mathbf{x}_p$ [S1]:
    $$ \varphi_p \left( \mathbf{x}' \right) = \mathbf{x}' \left( \mathbf{x}_p^\top \mathbf{x}' \right)^{-1} $$
    Note that $\mathbf{x}_p^\top \mathbf{x}_p = 1$, so the term $(\mathbf{x}_p^\top \mathbf{x}_p)$ in the source derivation simplifies to 1 [S1].

5.  **Compute Ray Direction and Project to 2D:**
    For a pixel $(u, v)$, construct the ray direction $\mathbf{d}$ in camera space using pinhole camera intrinsics [S1]:
    $$ \mathbf{d} = \begin{bmatrix} (u - c_x) / f_x \\ (v - c_y) / f_y \\ 1 \end{bmatrix} $$
    Project this ray direction onto the tangent plane to get the 2D coordinate $\mathbf{x}_{2D}$ [S1]:
    $$ \mathbf{x}_{2D} = \varphi_p \left( \mathbf{d} \right) = \mathbf{d} \left( \mathbf{x}_p^\top \mathbf{d} \right)^{-1} $$
    [S1]

6.  **Compute Jacobian Matrix $\mathbf{J}_p$:**
    The Jacobian of the projection function $\varphi_p$ at the mean $\boldsymbol{\mu}'$ is required to transform the 3D covariance $\boldsymbol{\Sigma}'$ (in camera space) to 2D [S1]. The general form is:
    $$ \mathbf{J}_p = \frac{\partial \varphi_p}{\partial \mathbf{x}'} \left( \boldsymbol{\mu}' \right) = \mathbb{I} \left( \mathbf{x}_p^\top \boldsymbol{\mu}' \right)^{-1} - \boldsymbol{\mu}' \left( \mathbf{x}_p^\top \boldsymbol{\mu}' \right)^{-1} \left( \boldsymbol{\mu}'^\top \mathbf{x}_p \right)^{-1} \mathbf{x}_p^\top $$
    Since $\mathbf{x}_p$ is parallel to $\boldsymbol{\mu}'$, $\mathbf{x}_p^\top \boldsymbol{\mu}' = \sqrt{N}$ [S1]. The explicit $3 \times 3$ matrix form for $\boldsymbol{\mu}' = [\mu_x', \mu_y', \mu_z']^\top$ is:
    $$ \mathbf{J}_p = \frac{1}{N^{3/2}} \begin{bmatrix} \mu_y'^2 + \mu_z'^2 & -\mu_x' \mu_y' & -\mu_x' \mu_z' \\ -\mu_x' \mu_y' & \mu_x'^2 + \mu_z'^2 & -\mu_y' \mu_z' \\ -\mu_x' \mu_z' & -\mu_y' \mu_z' & \mu_x'^2 + \mu_y'^2 \end{bmatrix} $$
    [S1]

7.  **Handle Rank Deficiency with Matrix $\mathbf{Q}$:**
    Both $\mathbf{J}_p$ and the original 3DGS Jacobian $\mathbf{J}$ have a rank of 2 [S1]. In standard 3DGS, the third row/column of $\mathbf{J} \boldsymbol{\Sigma}' \mathbf{J}^\top$ are zero, allowing simple truncation [S1]. However, for $\mathbf{J}_p \boldsymbol{\Sigma}' \mathbf{J}_p^\top$, the third row and column are not entirely zero [S1]. To ensure the 2D Gaussian equation holds, an invertible matrix $\mathbf{Q}$ is used for left multiplication with $\mathbf{x}_{2D}$, $\varphi_p(\boldsymbol{\mu}')$, and $\mathbf{J}_p$ [S1].
    
    The matrix $\mathbf{Q}$ is defined in the source as a product of two vectors, though the source's Equation 18 contains a dimensional inconsistency (the first vector has 3 components, the second has 4) [S1]. A robust implementation constructs $\mathbf{Q}$ to map the 3D tangent space to the 2D image plane consistently [G].

8.  **Compute 2D Gaussian Value:**
    The 2D Gaussian value $G_{2D}(\mathbf{x}_{2D})$ for alpha blending is computed as [S1]:
    $$ G_{2D}(\mathbf{x}_{2D}) = \exp \left\{ -\frac{1}{2} \left( \mathbf{x}_{2D} - \varphi_p(\boldsymbol{\mu}') \right)^T \left( \mathbf{J}_p \boldsymbol{\Sigma}' \mathbf{J}_p^\top \right)^{-1}_{2D} \left( \mathbf{x}_{2D} - \varphi_p(\boldsymbol{\mu}') \right) \right\} $$
    where $\left( \cdot \right)_{2D}$ denotes the inverse of the $2 \times 2$ submatrix obtained by skipping the third row and column of $\mathbf{J}_p \boldsymbol{\Sigma}' \mathbf{J}_p^\top$, adjusted by $\mathbf{Q}$ if necessary to handle the non-zero third row/column issue [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def optimal_gaussian_splatting(mu_world, sigma_world, W, cx, cy, fx, fy, u, v):
    """
    mu_world: (N, 3) world coordinates of Gaussian means
    sigma_world: (N, 3, 3) world covariance matrices
    W: (4, 4) world-to-camera transformation matrix
    cx, cy, fx, fy: camera intrinsics
    u, v: pixel coordinates (N,)
    """
    # 1. Transform to Camera Space
    mu_world_h = torch.cat([mu_world, torch.ones_like(mu_world[:, :1])], dim=1) # (N, 4)
    mu_cam_h = torch.matmul(mu_world_h, W.t()) # (N, 4)
    mu_prime = mu_cam_h[:, :3] # (N, 3) camera space mean
    
    # Transform covariance (rotation/scaling part of W)
    R_cam = W[:3, :3]
    sigma_prime = torch.bmm(torch.bmm(R_cam, sigma_world), R_cam.transpose(1, 2))
    
    # 2. Compute Projection Point on Unit Sphere x_p
    norm_sq = torch.sum(mu_prime**2, dim=1, keepdim=True) # (N, 1)
    norm_inv_sqrt = 1.0 / torch.sqrt(norm_sq) # (N, 1)
    x_p = mu_prime * norm_inv_sqrt # (N, 3)
    
    # 3. Compute Ray Direction d in Camera Space
    d_x = (u - cx) / fx
    d_y = (v - cy) / fy
    d = torch.stack([d_x, d_y, torch.ones_like(u)], dim=1) # (N, 3)
    
    # 4. Project Ray Direction to Tangent Plane: x_2D = d * (x_p^T * d)^-1
    dot_xp_d = torch.sum(x_p * d, dim=1, keepdim=True) # (N, 1)
    x_2D = d / dot_xp_d # (N, 3)
    
    # 5. Compute Jacobian J_p
    # J_p = (1/sqrt(N)) * I - (1/N) * (mu_prime * x_p^T)
    s = torch.sqrt(norm_sq) # (N, 1)
    outer_mu_xp = torch.bmm(mu_prime.unsqueeze(2), x_p.unsqueeze(1)) # (N, 3, 3)
    I = torch.eye(3).unsqueeze(0).expand(mu_prime.shape[0], -1, -1) # (N, 3, 3)
    
    J_p = (I / s.unsqueeze(2).unsqueeze(3)) - (outer_mu_xp / (s.unsqueeze(2).unsqueeze(3)**2))
    
    # 6. Compute 2D Covariance Sigma_2D
    Sigma_2D_3x3 = torch.bmm(torch.bmm(J_p, sigma_prime), J_p.transpose(1, 2))
    
    # 7. Handle Rank Deficiency
    # Source states to skip the third row and column. The Q matrix definition in [S1] Eq 18 
    # has a dimensional mismatch (3 vs 4 components), so we implement the standard truncation.
    Sigma_2D = Sigma_2D_3x3[:, :2, :2] # (N, 2, 2)
    
    # 8. Compute 2D Gaussian Value
    # phi_p(mu') = mu' / sqrt(N)
    mu_2D_proj = mu_prime / s.unsqueeze(1) # (N, 3)
    diff = x_2D - mu_2D_proj # (N, 3)
    diff_2D = diff[:, :2] # (N, 2)
    
    # Inverse of 2x2 covariance with numerical stability guard
    Sigma_2D_inv = torch.inverse(Sigma_2D + 1e-6 * torch.eye(2).unsqueeze(0))
    
    # Exponent: -0.5 * diff_2D^T @ Sigma_2D_inv @ diff_2D
    exponent = -0.5 * torch.sum((torch.bmm(diff_2D.unsqueeze(1), Sigma_2D_inv) * diff_2D.unsqueeze(2)).squeeze(1), dim=1)
    
    G_2D = torch.exp(exponent)
    
    return G_2D, x_2D, Sigma_2D
```

**Edge Cases and Pitfalls:**
*   Precision and boundaries: The norm $N$ must be checked against zero to avoid division by zero when computing $\mathbf{x}_p$ and $\mathbf{J}_p$ [S1].
*   Rank deficiency: The third row and column of $\mathbf{J}_p \boldsymbol{\Sigma}' \mathbf{J}_p^\top$ are not zero, requiring careful truncation or application of $\mathbf{Q}$ to maintain equation validity [S1].
*   Source ambiguity: The definition of $\mathbf{Q}$ in the source contains a dimensional inconsistency; implementers should resolve this by aligning the tangent plane basis with the 2D image plane [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method avoids the extensive ray sampling of NeRF, maintaining high performance [S1]. It is implemented with custom CUDA kernels for efficiency [S1].
*   **Strengths:** Reduces projection error, leading to higher-quality renderings [S1].
*   **Limitations:** The rank-2 handling with matrix $\mathbf{Q}$ is complex and prone to implementation errors due to ambiguities in the source formula [S1].

## Background
Optimal Gaussian Splatting was introduced in the paper "On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy" [S1]. It addresses the projection error inherent in the original 3D Gaussian Splatting method, which projects all Gaussians onto a single plane $z=1$ [S1]. The method is based on theoretical error analysis showing that the error is minimized when the projection of the Gaussian mean coincides with the ray direction from the camera center [S1]. It is implemented based on the PyTorch framework and 3D-GS with custom CUDA kernels [S1].

## Sources
[S1] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
