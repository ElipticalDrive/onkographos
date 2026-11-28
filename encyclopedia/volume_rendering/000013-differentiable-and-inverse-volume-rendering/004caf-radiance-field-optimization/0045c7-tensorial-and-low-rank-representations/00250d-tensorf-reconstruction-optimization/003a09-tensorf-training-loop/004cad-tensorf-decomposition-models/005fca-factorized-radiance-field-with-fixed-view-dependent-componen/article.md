# Factorized Radiance Field with Fixed View-Dependent Components

Also called: Point Cloud Fusion NeRF, VM Decomposition NeRF

## Overview
This method reconstructs a radiance field from sparse input views by factorizing geometry and appearance tensors into a fixed number of components equal to the number of input views ($n$), rather than a small fixed constant [S1]. It differs from TensoRF by binding each tensor component to a specific input view, enabling robust point cloud fusion that substitutes inaccurate depth-derived points with accurate ones from other views [S1]. The approach uses Vector-Matrix (VM) decomposition for spatial factors and a global appearance dictionary to abstract commonalities across the scene [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   $n$ input views, each containing an RGB image, a depth map, and camera parameters (intrinsic matrix $K$, rotation matrix $R$, translation vector $t$) [S1].
*   Hyperparameters: $\lambda_{depth}$ (balance between color and depth loss), $\omega_{reg}$ (L1 regularization weight), $P$ (feature decoding MLP dimension, set to 27), $N_0$ (initial grid size, set to 128), $N$ (final grid size), $Q$ (number of shading points per ray) [S1].

**Outputs:**
*   Rendered RGB images and depth maps for novel viewpoints [S1].

### Data Structures

**1. Geometry Grid ($G_\sigma$)**
A factorized 3D tensor representing volume density $\sigma$ [S1]. It is decomposed into $n$ components, one per input view [S1]. Each component $r$ ($1 \le r \le n$) consists of:
*   Three 1D vectors: $v_{\sigma,r}^X \in \mathbb{R}^I$, $v_{\sigma,r}^Y \in \mathbb{R}^J$, $v_{\sigma,r}^Z \in \mathbb{R}^K$ [S1].
*   Three 2D matrices: $M_{\sigma,r}^{XY} \in \mathbb{R}^{I \times J}$, $M_{\sigma,r}^{XZ} \in \mathbb{R}^{I \times K}$, $M_{\sigma,r}^{YZ} \in \mathbb{R}^{J \times K}$ [S1].
*   The grid dimensions are $I \times J \times K$ [S1].

**2. Appearance Grid ($G_c$)**
A factorized 3D tensor representing color $c$ [S1]. It is also decomposed into $n$ components [S1]. Each component $r$ involves:
*   Three 1D vectors: $v_{c,r}^X \in \mathbb{R}^I$, $v_{c,r}^Y \in \mathbb{R}^J$, $v_{c,r}^Z \in \mathbb{R}^K$ [S1].
*   Three 2D matrices: $A_{c,r}^X \in \mathbb{R}^{I \times J}$, $A_{c,r}^Y \in \mathbb{R}^{J \times K}$, $A_{c,r}^Z \in \mathbb{R}^{I \times K}$ [S1].
*   Three basis vectors: $b_{3r-2}, b_{3r-1}, b_{3r} \in \mathbb{R}^P$ [S1].

**3. Global Appearance Dictionary ($B$)**
A matrix of size $P \times 3n$ formed by stacking the $3n$ basis vectors $b_r$ [S1].

### Initialization

1.  **NDC Mapping:** Map depth values to Normalized Device Coordinate (NDC) space to ensure all visible locations fit within a predetermined cubic space [S1].
2.  **Point Cloud Generation:** For each input view $r$, map every 2D pixel location $x_p$ to a 3D world location $x_w$ using the camera parameters:
    $$x_w = R^{-1} (K^{-1} x_p D - t)$$
    where $D$ is the depth value at $x_p$ [S1].
3.  **Grid Assignment:**
    *   **Geometry:** For each point $x_w$ from view $r$, identify the corresponding voxel indices $(i,j,k)$ in the $I \times J \times K$ grid [S1]. Set the elements of $v_{\sigma,r}^X, v_{\sigma,r}^Y, v_{\sigma,r}^Z$ and $M_{\sigma,r}^{XY}, M_{\sigma,r}^{XZ}, M_{\sigma,r}^{YZ}$ corresponding to these projections to 1 [S1]. All other elements are set to 0 [S1].
    *   **Appearance:** For the same projections, assign the average RGB color of the projected points to the elements of $A_{c,r}^X, A_{c,r}^Y, A_{c,r}^Z$ [S1]. Assign random values to $v_{c,r}^X, v_{c,r}^Y, v_{c,r}^Z$ [S1].
4.  **Dictionary Initialization:** Initialize the basis vectors $b_r$ (and thus matrix $B$) [S1].

### Rendering

To render a pixel, march along a ray and sample $Q$ shading points $x_q$ with step size $\Delta_q$ [S1].

1.  **Density Interpolation:** Calculate $\sigma_q = G_\sigma(x_q)$ using trilinear interpolation of the factorized geometry grid [S1]. For a single component $r$, the interpolated value follows $A_{\sigma,r}(x) = v_{\sigma,r}(x) M_{\sigma,r}(y, z)$ [S1].
2.  **Color Interpolation:** Calculate the feature vector $f_q$ by concatenating the interpolated appearance factors for component $r$, then multiplying by the dictionary $B$:
    $$f_q = B \oplus [A_{c,r}(x_q)]_{r}$$
    The color $c_q$ is obtained by passing $f_q$ through a feature decoding MLP $S$:
    $$c_q = S(f_q, d_q)$$
    where $d_q$ is the view direction at $x_q$ [S1].
3.  **Volume Rendering:** Compute the pixel color $C$ and transmittance $\tau_q$:
    $$\tau_q = \exp\left(-\sum_{p=1}^{q-1} \sigma_p \Delta_p\right)$$
    $$C = \sum_{q=1}^{Q} \tau_q (1 - \exp(-\sigma_q \Delta_q)) c_q$$
    $$\tau_{Q+1} = \exp\left(-\sum_{p=1}^{Q} \sigma_p \Delta_p\right)$$
    [S1].

### Optimization

The model is optimized using a composite loss function $L$:
$$L = L_{RGB} + \lambda_{depth} L_{depth}$$

1.  **RGB Loss ($L_{RGB}$):**
    $$L_{RGB} = \|C - \hat{C}\|_2^2 + \omega_{reg} L_{reg}$$
    where $\hat{C}$ is the ground truth color, $C$ is the predicted color, and $L_{reg}$ is the L1 regularization term applied to all tensor factors to promote sparsity and eliminate floaters [S1].
2.  **Depth Loss ($L_{depth}$):**
    $$L_{depth} = \|D - \hat{D}\|_2^2$$
    where $D$ is the rendered depth and $\hat{D}$ is the ground truth depth [S1].

### Coarse-to-Fine Reconstruction

Training starts with a grid of size $N_0^3$ ($N_0=128$) [S1]. The grid is upsampled at specific training steps (2000, 3000, 4000, 5500, 7000) to the final size $N^3$ [S1]. Upsampling is performed by linearly and bilinearly upsampling the XYZ-mode vector and matrix factors [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def bilinear_interpolate(grid, x, y):
    """Interpolate 2D grid at continuous coordinates x, y in [0, W], [0, H]."""
    x0, x1 = torch.floor(x).long(), torch.ceil(x).long()
    y0, y1 = torch.floor(y).long(), torch.ceil(y).long()
    x0 = torch.clamp(x0, 0, grid.shape[1] - 1)
    x1 = torch.clamp(x1, 0, grid.shape[1] - 1)
    y0 = torch.clamp(y0, 0, grid.shape[2] - 1)
    y1 = torch.clamp(y1, 0, grid.shape[2] - 1)
    
    wa = (x1.float() - x)
    wb = (x - x0.float())
    hc = (y1.float() - y)
    hd = (y - y0.float())
    
    val = (wa * hc * grid[:, x0, y0] +
           wb * hc * grid[:, x1, y0] +
           wa * hd * grid[:, x0, y1] +
           wb * hd * grid[:, x1, y1])
    return val

def trilinear_interpolate(grid, x, y, z):
    """Interpolate 3D grid at continuous coordinates x, y, z in [0, D], [0, H], [0, W]."""
    x0, x1 = torch.floor(x).long(), torch.ceil(x).long()
    y0, y1 = torch.floor(y).long(), torch.ceil(y).long()
    z0, z1 = torch.floor(z).long(), torch.ceil(z).long()
    x0 = torch.clamp(x0, 0, grid.shape[1] - 1)
    x1 = torch.clamp(x1, 0, grid.shape[1] - 1)
    y0 = torch.clamp(y0, 0, grid.shape[2] - 1)
    y1 = torch.clamp(y1, 0, grid.shape[2] - 1)
    z0 = torch.clamp(z0, 0, grid.shape[3] - 1)
    z1 = torch.clamp(z1, 0, grid.shape[3] - 1)
    
    wa, wb = (x1.float() - x), (x - x0.float())
    hc, hd = (y1.float() - y), (y - y0.float())
    sc, sd = (z1.float() - z), (z - z0.float())
    
    val = (wa * hc * sc * grid[:, x0, y0, z0] +
           wb * hc * sc * grid[:, x1, y0, z0] +
           wa * hd * sc * grid[:, x0, y1, z0] +
           wb * hd * sc * grid[:, x1, y1, z0] +
           wa * hc * sd * grid[:, x0, y0, z1] +
           wb * hc * sd * grid[:, x1, y0, z1] +
           wa * hd * sd * grid[:, x0, y1, z1] +
           wb * hd * sd * grid[:, x1, y1, z1])
    return val

class FactorizedRadianceField:
    def __init__(self, n_views, I, J, K, P, N0=128):
        self.n = n_views
        self.I, self.J, self.K = I, J, K
        self.P = P
        
        # Geometry Grid Factors: n components
        self.v_sigma_X = nn.Parameter(torch.randn(n_views, I))
        self.v_sigma_Y = nn.Parameter(torch.randn(n_views, J))
        self.v_sigma_Z = nn.Parameter(torch.randn(n_views, K))
        
        self.M_sigma_XY = nn.Parameter(torch.randn(n_views, I, J))
        self.M_sigma_XZ = nn.Parameter(torch.randn(n_views, I, K))
        self.M_sigma_YZ = nn.Parameter(torch.randn(n_views, J, K))
        
        # Appearance Grid Factors: n components
        self.v_c_X = nn.Parameter(torch.randn(n_views, I))
        self.v_c_Y = nn.Parameter(torch.randn(n_views, J))
        self.v_c_Z = nn.Parameter(torch.randn(n_views, K))
        
        self.A_c_X = nn.Parameter(torch.randn(n_views, I, J))
        self.A_c_Y = nn.Parameter(torch.randn(n_views, J, K))
        self.A_c_Z = nn.Parameter(torch.randn(n_views, I, K))
        
        # Global Appearance Dictionary B: P x 3n
        self.B = nn.Parameter(torch.randn(P, 3 * n_views))
        
        # Feature Decoding MLP
        self.mlp = nn.Sequential(
            nn.Linear(P, 64),
            nn.ReLU(),
            nn.Linear(64, 3)
        )
        
        self.lambda_depth = 1.0
        self.omega_reg = 1e-4

    def get_density(self, x, y, z):
        # Interpolate vectors
        v_x = trilinear_interpolate(self.v_sigma_X.unsqueeze(1).unsqueeze(1), x, y, z)
        v_y = trilinear_interpolate(self.v_sigma_Y.unsqueeze(1).unsqueeze(1), x, y, z)
        v_z = trilinear_interpolate(self.v_sigma_Z.unsqueeze(1).unsqueeze(1), x, y, z)
        
        # Interpolate matrices
        M_yz = bilinear_interpolate(self.M_sigma_YZ, y, z)
        M_xz = bilinear_interpolate(self.M_sigma_XZ, x, z)
        M_xy = bilinear_interpolate(self.M_sigma_XY, x, y)
        
        # VM decomposition sum: sigma = sum_r (v_X * M_YZ + v_Y * M_XZ + v_Z * M_XY)
        sigma = torch.sum(v_x * M_yz + v_y * M_xz + v_z * M_xy, dim=0)
        return sigma

    def get_color(self, x, y, z, d):
        # Interpolate appearance matrices
        A_yz = bilinear_interpolate(self.A_c_Y, y, z)
        A_xz = bilinear_interpolate(self.A_c_Z, x, z)
        A_xy = bilinear_interpolate(self.A_c_X, x, y)
        
        # Concatenate features for each view r (3 matrices per view -> 3n features)
        features = torch.cat([A_xy.flatten(1), A_yz.flatten(1), A_xz.flatten(1)], dim=1)
        
        # Multiply by Dictionary B
        features = torch.matmul(features, self.B.T)
        
        # Pass through MLP
        colors = self.mlp(features)
        return colors

    def render_ray(self, ray_origins, ray_directions, t_near, t_far, Q=128):
        # Sample points along ray
        t_vals = torch.linspace(t_near, t_far, Q).unsqueeze(0)
        t_vals = t_vals.expand(ray_origins.shape[0], -1)
        points = ray_origins.unsqueeze(1) + t_vals.unsqueeze(2) * ray_directions.unsqueeze(1)
        
        # Map NDC world coordinates to grid coordinates [0, I], [0, J], [0, K]
        # Assuming points are in NDC [-1, 1]^3
        x_grid = (points[:, :, 0] + 1) * 0.5 * self.I
        y_grid = (points[:, :, 1] + 1) * 0.5 * self.J
        z_grid = (points[:, :, 2] + 1) * 0.5 * self.K
        
        sigma = self.get_density(x_grid, y_grid, z_grid)
        colors = self.get_color(x_grid, y_grid, z_grid, ray_directions)
        
        # Volume rendering
        deltas = t_vals[:, 1:] - t_vals[:, :-1]
        deltas = torch.cat([deltas, torch.zeros_like(deltas[:, :1])], dim=1)
        
        alpha = 1.0 - torch.exp(-sigma * deltas)
        T = torch.cumprod(torch.exp(-sigma * deltas), dim=1)
        T = torch.cat([torch.ones_like(T[:, :1]), 1.0 - T], dim=1)
        
        weights = alpha * T[:, :-1]
        C = torch.sum(weights.unsqueeze(2) * colors[:, :-1], dim=1)
        
        return C

    def loss(self, pred_color, pred_depth, gt_color, gt_depth):
        L_rgb = torch.mean((pred_color - gt_color)**2) + self.omega_reg * torch.mean(torch.abs(self.v_sigma_X))
        L_depth = torch.mean((pred_depth - gt_depth)**2)
        return L_rgb + self.lambda_depth * L_depth
```

## Operating characteristics

**Time and Memory Complexity:**
*   Memory: The number of tensor components is fixed to the number of input views $n$ [S1]. Memory scales with $n \times (I \cdot J + J \cdot K + I \cdot K)$ for geometry and appearance matrices, plus $P \times 3n$ for the dictionary [S1].
*   Time: Rendering involves interpolating factors for each sample point. The number of components $n$ adds a summation loop during density/color calculation [S1].

**Measured Performance:**
*   **LLFF Dataset (2 views):** PSNR 17.83, SSIM 0.5512, LPIPS 0.3832 [S1].
*   **LLFF Dataset (4 views):** PSNR 20.86, SSIM 0.5967, LPIPS 0.3247 [S1].
*   **DTU Dataset (0% noise):** PSNR 19.34, SSIM 0.7431, LPIPS 0.2576 [S1].
*   **DTU Dataset (10% depth noise):** PSNR 16.63, SSIM 0.4144, LPIPS 0.5561 [S1].

**Strengths and Limitations:**
*   **Strengths:** Robust to inaccurate depth values through point cloud fusion; eliminates floaters via L1 regularization; performs well on sparse views [S1].
*   **Limitations:** Performance degrades with noisy depth maps (e.g., 10% noise significantly reduces PSNR) [S1].

**When to Use:**
*   Sparse view reconstruction where depth maps are available but potentially noisy [S1].
*   Scenes where view-dependent appearance can be abstracted into a global dictionary [S1].

## Background

**Origin:**
The method was introduced in the paper "Depth-Guided Robust and Fast Point Cloud Fusion NeRF for Sparse Input Views" presented at the Thirty-Eighth AAAI Conference on Artificial Intelligence (AAAI-24) [S1].

**History and Development:**
It builds upon TensoRF but modifies the decomposition to fix the number of components to the number of input views, enabling explicit point cloud fusion [S1]. It uses NDC normalization similar to DSNeRF [S1].

**Usage:**
Implemented in PyTorch, optimized on NVIDIA RTX 4090 GPU with a batch size of 4096 pixel rays [S1].

## Sources

[S1] Depth-Guided Robust and Fast Point Cloud Fusion NeRF for Sparse Input Views, https://doi.org/10.1609/aaai.v38i3.27968, fetched 2026-10-29
