# 3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)

## Overview
3iGS enhances 3D Gaussian Splatting by modeling outgoing radiance as a function of a continuous local illumination field and individual Gaussian BRDF attributes, rather than optimizing outgoing radiance in isolation [S1]. It differs from its sibling [SWAG](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md) by using a factorised tensor grid for local illumination and a neural network for specular shading, whereas SWAG conditions Gaussians on external appearance variables like image ID or camera pose [S1].

## How it works
**Inputs:** A set of 3D Gaussians defined by mean $\boldsymbol{\mu}_i$, opacity $\alpha_i$, scaling matrix $\mathbf{S}_i$, rotation matrix $\mathbf{R}_i$, diffuse color $\mathbf{c}_{d,i}$, and BRDF features $\boldsymbol{\rho}_i$. A learnable factorised tensor illumination grid $\mathcal{G}_l$. A neural network $\mathcal{F}$ for specular shading.
**Outputs:** Rendered image $\hat{\mathbf{C}}$.

**Data Structures:**
1.  **Illumination Grid $\mathcal{G}_l$:** Represented using vector-matrix spatial factorisation. It consists of feature tensors $\mathbf{A}_{L,r}^X, \mathbf{A}_{L,r}^Y, \mathbf{A}_{L,r}^Z$ and feature vectors $\mathbf{b}_k$. The grid has a coarse resolution of $150^3$ voxels [S1].
2.  **Gaussian Attributes:**
    *   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Mean coordinates.
    *   $\alpha_i \in \mathbb{R}$: Opacity.
    *   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scaling matrix (diagonal).
    *   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix (orthogonal).
    *   $\mathbf{c}_{d,i} \in \mathbb{R}^3$: Diffuse color.
    *   $\boldsymbol{\rho}_i$: BRDF features. These are repurposed from the standard 3DGS Spherical Harmonics (SH) coefficients. Specifically, the 16x3 SH coefficients are used as BRDF feature channels, plus 4 additional parameters for base colour and roughness used in Integrated Directional Encoding (IDE) [S1].

**Numbered Steps:**

1.  **Covariance Transformation:** For each Gaussian $i$, compute the 3D covariance matrix $\boldsymbol{\Sigma}_i$ from the learned scaling $\mathbf{S}_i$ and rotation $\mathbf{R}_i$:
    $$ \boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T $$
    Transform $\boldsymbol{\Sigma}_i$ to camera coordinates using the viewing transformation matrix $\mathbf{W}$ and the affine approximation of the projective transformation $\mathbf{J}$:
    $$ \boldsymbol{\Sigma}'_i = \mathbf{J} \mathbf{W} \boldsymbol{\Sigma}_i \mathbf{W}^T \mathbf{J}^T $$
    [S1]

2.  **Illumination Query:** For each Gaussian $i$ at mean coordinate $\mathbf{x}_i$ (which is $\boldsymbol{\mu}_i$), query the illumination grid $\mathcal{G}_l$ to obtain the incoming illumination $L_i$. The grid is defined as:
    $$ \mathcal{G}_l = \sum_{r=1}^{R_L} \mathbf{A}_{L,r}^X \circ \mathbf{b}_{3r-2} + \mathbf{A}_{L,r}^Y \circ \mathbf{b}_{3r-1} + \mathbf{A}_{L,r}^Z \circ \mathbf{b}_{3r} $$
    where $R_L$ is the number of feature channels, $\circ$ denotes the outer product or appropriate tensor contraction, and $\mathbf{b}$ are the feature vectors. Interpolation is performed from the factorised tensors [S1].

3.  **Specular Shading:** Compute the specular color $\mathbf{c}_{s,i}$ for viewing direction $\omega_o$ using the neural network $\mathcal{F}$:
    $$ \mathcal{F} : \{ \boldsymbol{\rho}_i, L_i, \omega_o \} \mapsto \mathbf{c}_{s,i} $$
    The viewing direction $\omega_o$ is encoded using Integrated Directional Encoding (IDE) [S1].

4.  **Radiance Composition:** Compute the total outgoing radiance $\mathbf{c}_i(\omega_o)$ by linearly adding the diffuse and specular components:
    $$ \mathbf{c}_i(\omega_o) = \mathbf{c}_{d,i} + \mathbf{c}_{s,i} $$
    [S1]

5.  **Rasterization:** Render the scene using the standard 3DGS alpha-blending pipeline. For a pixel $\mathbf{u}$, the color $\hat{\mathbf{C}}$ is:
    $$ \hat{\mathbf{C}} = \sum_{i \in N} T_i g_i(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') \alpha_i \mathbf{c}_i(\omega_o) $$
    where $T_i = \prod_{j=1}^{i-1} (1 - g_j(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') \alpha_j)$ is the transmittance, and $g_i$ is the 2D Gaussian density:
    $$ g_i(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') = e^{-\frac{1}{2}(\mathbf{u} - \boldsymbol{\mu}')^T (\boldsymbol{\Sigma}')^{-1} (\mathbf{u} - \boldsymbol{\mu}')} $$
    [S1]

**Optimization Strategy:**
*   **Phase 1 (Iterations 0–3,000):** Train the model with only the diffuse color $\mathbf{c}_{d,i}$. Optimize $\boldsymbol{\mu}_i, \alpha_i, \mathbf{S}_i, \mathbf{R}_i, \mathbf{c}_{d,i}$ and the illumination grid parameters [S1].
*   **Phase 2 (Iterations 3,000–30,000):** Add specular colors. Optimize $\boldsymbol{\rho}_i$ (BRDF features) alongside other parameters [S1].
*   **Grid Boundary Management:** Define an initial boundary encapsulating the scene bounding box. Midway through training, shrink the illumination grid to fit the Gaussians and resample the grid with the same number of voxels [S1].
*   **Loss Function:**
    $$ \mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda \mathcal{L}_{\text{D-SSIM}} $$
    with $\lambda = 0.2$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FactorisedTensorGrid(nn.Module):
    """
    Represents the illumination grid G_l using vector-matrix factorisation.
    """
    def __init__(self, resolution=150, rank=16, channels=3):
        super().__init__()
        self.rank = rank
        self.channels = channels
        # A_X, A_Y, A_Z are tensors of shape [rank, resolution, channels]
        # b are vectors of shape [rank * 3]
        self.A_X = nn.Parameter(torch.randn(rank, resolution, channels))
        self.A_Y = nn.Parameter(torch.randn(rank, resolution, channels))
        self.A_Z = nn.Parameter(torch.randn(rank, resolution, channels))
        self.b = nn.Parameter(torch.randn(rank * 3))

    def forward(self, coords):
        """
        Query the grid at normalized coordinates coords (N, 3) in [0, 1].
        Returns illumination features (N, channels).
        """
        # Interpolation logic for factorised tensors is complex.
        # Standard TensoRF interpolation involves trilinear interpolation of the factor matrices.
        # For implementation, one typically interpolates the factor matrices A_X, A_Y, A_Z
        # at the projected coordinates and combines them with b.
        # This is a placeholder for the specific interpolation math defined in [S1] Eq 11.
        # In practice, this calls the interpolation routine from the TensoRF library or equivalent.
        # Here we assume a helper function `interpolate_factorised_grid` exists.
        return interpolate_factorised_grid(self.A_X, self.A_Y, self.A_Z, self.b, coords)

class SpecularNetwork(nn.Module):
    """
    Neural network F mapping BRDF features, illumination, and view direction to specular color.
    """
    def __init__(self, in_dim=20, hidden_dim=64):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(in_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 3) # RGB output
        )

    def forward(self, brdf_features, illumination, view_dirs):
        # Concatenate inputs
        x = torch.cat([brdf_features, illumination, view_dirs], dim=-1)
        return self.net(x)

def render_gaussians(gaussians, camera_params, grid, specular_net):
    """
    Render a batch of Gaussians.
    gaussians: dict with keys 'mu', 'alpha', 'S', 'R', 'c_d', 'rho'
    camera_params: dict with 'W', 'J'
    grid: FactorisedTensorGrid instance
    specular_net: SpecularNetwork instance
    """
    mu = gaussians['mu']
    alpha = gaussians['alpha']
    S = gaussians['S']
    R = gaussians['R']
    c_d = gaussians['c_d']
    rho = gaussians['rho']
    
    W = camera_params['W']
    J = camera_params['J']
    
    # 1. Covariance Transformation
    # Sigma = R S S^T R^T
    S_T = S.transpose(-2, -1)
    Sigma = R @ S @ S_T @ R_T
    
    # Sigma' = J W Sigma W^T J^T
    Sigma_prime = J @ W @ Sigma @ W.transpose(-2, -1) @ J.transpose(-2, -1)
    
    # 2. Illumination Query
    # Query grid at mu
    L_i = grid(mu)
    
    # 3. Specular Shading
    # View directions omega_o need to be computed for each pixel/Gaussian intersection.
    # In rasterization, this is done per-pixel. Here we assume view_dirs are passed or computed.
    # For simplicity, assume view_dirs is (N, 3) normalized vectors from camera to mu.
    # Note: In actual rasterization, view_dirs vary per pixel.
    # c_s = F(rho, L_i, omega_o)
    # c_s = specular_net(rho, L_i, view_dirs)
    
    # 4. Radiance Composition
    # c = c_d + c_s
    # c_total = c_d + c_s
    
    # 5. Rasterization
    # Standard 3DGS rasterization loop
    # ...
    pass

def train_step(gaussians, grid, specular_net, images, camera_params, iteration):
    """
    Single training step.
    """
    # Phase 1: 0-3000 iterations, only diffuse
    # Phase 2: 3000-30000 iterations, diffuse + specular
    
    if iteration < 3000:
        # Disable specular network gradients or ignore specular loss
        c_total = gaussians['c_d']
    else:
        # Compute specular
        # Note: view_dirs must be computed based on camera and gaussian positions
        view_dirs = compute_view_dirs(gaussians['mu'], camera_params['origin'])
        L_i = grid(gaussians['mu'])
        c_s = specular_net(gaussians['rho'], L_i, view_dirs)
        c_total = gaussians['c_d'] + c_s
        
    # Render
    rendered_image = render_gaussians({
        'mu': gaussians['mu'],
        'alpha': gaussians['alpha'],
        'S': gaussians['S'],
        'R': gaussians['R'],
        'c_d': gaussians['c_d'],
        'rho': gaussians['rho']
    }, camera_params, grid, specular_net)
    
    # Loss
    L1 = F.l1_loss(rendered_image, images)
    D_SSIM = compute_dssim(rendered_image, images)
    lambda_val = 0.2
    loss = (1 - lambda_val) * L1 + lambda_val * D_SSIM
    
    loss.backward()
    return loss
```

**Edge Cases and Pitfalls:**
*   **Grid Boundary:** The illumination grid must be managed carefully. Shrinking the grid midway through training requires resampling to maintain voxel count [S1].
*   **BRDF Features:** The BRDF features $\boldsymbol{\rho}_i$ are repurposed from SH coefficients. This requires careful initialization and optimization to avoid conflicts with the diffuse color representation [S1].
*   **Precision:** The factorised tensor representation relies on interpolation. Ensure interpolation is smooth to avoid artifacts in the illumination field [S1].

## Operating characteristics
*   **Time Complexity:** Rendering speed is comparable to 3DGS because the illumination field is pre-calculated and stored in volumes, allowing fast evaluation [S1]. The neural network $\mathcal{F}$ adds some overhead per Gaussian but is lightweight.
*   **Memory Complexity:** The illumination field uses a coarse resolution of $150^3$ voxels, which is 87.5% less memory than TensoRF on synthetic datasets [S1]. The memory footprint is low compared to the number of optimized Gaussians [S1].
*   **Performance:** On the Shiny Blender dataset, 3iGS achieves an average PSNR of 27.51, SSIM of 0.930, and LPIPS of 0.045 [S1]. It surpasses 3DGS in scenes with reflective surfaces and specular view-dependent effects [S1].
*   **Strengths:** Enhances rendering quality of specular effects; maintains rapid training and rendering speeds; compact memory footprint for illumination [S1].
*   **Limitations:** Requires additional optimization of BRDF features and illumination grid; training strategy is split into two phases [S1].

## Background
3iGS was introduced by Z.J. Tang and T.J. Cham [S1]. It builds on 3D Gaussian Splatting (3DGS) [S1] and addresses its limitations in modeling view-dependent effects and specular highlights by incorporating a local illumination field and BRDF attributes [S1]. It differs from GaussianShader [13] by using grid-based tensors for local illumination instead of global environment lights [S1].

## Sources
[S1] 3iGS: Factorised Tensorial Illumination for 3D Gaussian Splatting, https://core.ac.uk/works/391741134, fetched 2026-10-29
