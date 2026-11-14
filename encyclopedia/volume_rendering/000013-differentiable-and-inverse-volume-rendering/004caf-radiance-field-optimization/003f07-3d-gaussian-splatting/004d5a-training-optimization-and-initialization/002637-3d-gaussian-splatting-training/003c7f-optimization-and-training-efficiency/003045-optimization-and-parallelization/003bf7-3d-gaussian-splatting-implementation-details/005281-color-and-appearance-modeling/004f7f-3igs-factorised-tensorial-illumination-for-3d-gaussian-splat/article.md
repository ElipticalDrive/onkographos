# 3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)


## Overview
3iGS enhances 3D Gaussian Splatting (3DGS) by modeling outgoing radiance as a function of a continuous local illumination field and individual Gaussian BRDF attributes, rather than optimizing outgoing radiance in isolation [S1]. It differs from its sibling [SWAG](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md) by using a factorized tensor grid for local illumination rather than appearance-conditioned Gaussians for global lighting changes, and from [GaussianShader](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md) by using grid-based tensors instead of global environment maps [S1].

## How it works
**Inputs:** A set of 3D Gaussians (initialized from SfM points or random initialization), camera views, and ground truth images.
**Outputs:** Rendered images and optimized scene parameters (Gaussian attributes, illumination grid, neural network weights).

**Data Structures:**
1.  **3D Gaussians:** Each Gaussian $i$ is defined by:
    *   Mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$.
    *   Opacity $\alpha_i \in \mathbb{R}$.
    *   Scaling matrix $\mathbf{S}_i$ and Rotation matrix $\mathbf{R}_i$ (used to derive covariance $\boldsymbol{\Sigma}_i = \mathbf{R}\mathbf{S}\mathbf{S}^T\mathbf{R}^T$) [S1].
    *   Diffuse color $\mathbf{c}_d$ (scalar or vector).
    *   BRDF features $\boldsymbol{\rho}_i$: Repurposed 16x3 SH coefficients plus 4 additional parameters for base color and roughness [S1].
2.  **Illumination Grid $\mathcal{G}_l$:** A factorized tensor representation of the local illumination field.
    *   Resolution: Coarse grid of $150^3$ voxels [S1].
    *   Structure: Vector-matrix spatial factorization with $R_L$ feature channels. Composed of feature tensors $\mathbf{A}_{L,r}^X, \mathbf{A}_{L,r}^Y, \mathbf{A}_{L,r}^Z$ and feature vectors $\mathbf{b}$ [S1].
3.  **Neural Renderer $\mathcal{F}$:** A small MLP mapping features to specular color [S1].

**Numbered Steps:**

1.  **Initialize Gaussians:** Fit 3D Gaussians to a point cloud (SfM or random). Initialize parameters: $\boldsymbol{\mu}_i, \alpha_i, \mathbf{S}_i, \mathbf{R}_i, \mathbf{c}_d, \boldsymbol{\rho}_i$ [S1].
2.  **Initialize Illumination Grid:** Define an initial boundary encapsulating the scene bounding box. Initialize the factorized tensor grid $\mathcal{G}_l$ with resolution $150^3$ [S1].
3.  **Training Phase 1 (Diffuse Only):** For the first 3,000 iterations, optimize the model using only the diffuse color $\mathbf{c}_d$. Do not compute or optimize specular components [S1].
4.  **Training Phase 2 (Specular Addition):** After 3,000 iterations, enable specular color computation [S1].
5.  **Grid Boundary Management:** Midway through training, shrink the illumination grid boundary to fit the current distribution of Gaussians. Resample the grid to maintain the same number of voxels ($150^3$) [S1].
6.  **Compute Illumination $L_i$:** For each Gaussian $i$, query the illumination grid $\mathcal{G}_l$ using its mean coordinate $\mathbf{x}_i$ (derived from $\boldsymbol{\mu}_i$). Interpolate from the factorized tensors to obtain incoming illumination $L_i$:
    $$ \mathcal{G}_l : \{ \mathbf{x}_i \} \mapsto L_i $$
    The grid is formulated as:
    $$ \mathcal{G}_l = \sum_{r=1}^{R_L} \mathbf{A}_{L,r}^X \circ \mathbf{b}_{3r-2} + \mathbf{A}_{L,r}^Y \circ \mathbf{b}_{3r-1} + \mathbf{A}_{L,r}^Z \circ \mathbf{b}_{3r} $$
    where $\circ$ denotes the outer product or appropriate tensor contraction, and $\mathbf{b}$ are the feature vectors [S1].
7.  **Compute Specular Color $\mathbf{c}_s$:** Encode the viewing direction $\omega_o$ using Integrated Directional Encoding (IDE). Combine $\omega_o$, Gaussian BRDF features $\boldsymbol{\rho}_i$, and illumination $L_i$ in the neural network $\mathcal{F}$:
    $$ \mathcal{F} : \{ \boldsymbol{\rho}_i, L_i, \omega_o \} \mapsto \mathbf{c}_s $$
8.  **Compute Outgoing Radiance $\mathbf{c}$:** Sum the diffuse and specular components:
    $$ \mathbf{c}(\omega_o) = \mathbf{c}_d + \mathbf{c}_s(\omega_o) $$
9.  **Render:** Project Gaussians to 2D. The covariance in camera coordinates is $\boldsymbol{\Sigma}' = \mathbf{J}\mathbf{W}\boldsymbol{\Sigma}\mathbf{W}^T\mathbf{J}^T$, where $\mathbf{W}$ is the viewing transformation and $\mathbf{J}$ is the affine approximation of the projective transformation [S1]. Rasterize using alpha blending:
    $$ \hat{\mathbf{C}} = \sum_{i \in N} T_i g_i(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') \alpha_i \mathbf{c}_i, \quad T_i = \prod_{j=1}^{i-1} (1 - g_j(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') \alpha_j) $$
    where $g(\mathbf{x}|\boldsymbol{\mu}, \boldsymbol{\Sigma}) = e^{-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})}$ [S1].
10. **Loss Calculation:** Compute loss $\mathcal{L}$ against ground truth:
    $$ \mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda \mathcal{L}_{\text{D-SSIM}} $$
    with $\lambda = 0.2$ [S1].
11. **Optimization:** Update all parameters ($\boldsymbol{\mu}_i, \alpha_i, \mathbf{S}_i, \mathbf{R}_i, \mathbf{c}_d, \boldsymbol{\rho}_i, \mathcal{G}_l, \mathcal{F}$) via gradient descent. Total training iterations: 30,000 [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class FactorizedTensorGrid(nn.Module):
    """
    Represents the illumination grid G_l using vector-matrix factorization.
    """
    def __init__(self, resolution=150, rank=4): # Rank R_L not explicitly given, assume typical value or parameter
        super().__init__()
        self.resolution = resolution
        self.rank = rank
        # Initialize feature tensors A and vectors b
        # Dimensions depend on specific tensor factorization implementation details
        # Not fully specified in sources, assuming standard factorized grid structure
        self.A_X = nn.Parameter(torch.randn(rank, resolution, resolution, resolution))
        self.A_Y = nn.Parameter(torch.randn(rank, resolution, resolution, resolution))
        self.A_Z = nn.Parameter(torch.randn(rank, resolution, resolution, resolution))
        self.b = nn.Parameter(torch.randn(rank * 3)) # Feature vectors

    def forward(self, coords):
        """
        Query the grid at normalized coordinates coords (N, 3) in [0, 1].
        Returns illumination features L (N, channels).
        """
        # Interpolation from factorized tensors
        # Exact interpolation logic for factorized tensors (e.g., TensoRF style)
        # is complex and relies on external library or detailed implementation
        # not provided in [S1]. Placeholder for interpolation logic.
        # L = interpolate_factorized_tensors(self.A_X, self.A_Y, self.A_Z, self.b, coords)
        # [G] Standard tri-plane or factorized grid interpolation would be used here.
        # For the purpose of this article, we assume a function `query_factorized_grid` exists.
        L = query_factorized_grid(self.A_X, self.A_Y, self.A_Z, self.b, coords)
        return L

class NeuralRenderer(nn.Module):
    """
    Maps BRDF features, illumination, and view direction to specular color.
    """
    def __init__(self, input_dim, output_dim=3):
        super().__init__()
        self.mlp = nn.Sequential(
            nn.Linear(input_dim, 64),
            nn.ReLU(),
            nn.Linear(64, 64),
            nn.ReLU(),
            nn.Linear(64, output_dim)
        )

    def forward(self, brdf_features, illumination, view_dirs_encoded):
        """
        brdf_features: (N, D_rho)
        illumination: (N, D_L)
        view_dirs_encoded: (N, D_ide)
        """
        x = torch.cat([brdf_features, illumination, view_dirs_encoded], dim=-1)
        return self.mlp(x)

def render_gaussians(gaussians, camera_params, grid, renderer):
    """
    Renders the scene using 3DGS rasterization.
    """
    # 1. Project Gaussians to 2D
    # 2. Sort by depth
    # 3. Rasterize and blend
    # This part follows standard 3DGS implementation [S1]
    pass

def train_step(gaussians, camera, gt_image, grid, renderer, optimizer):
    # 1. Get Gaussian means
    means = gaussians['means']
    
    # 2. Query Illumination Grid
    # Normalize means to grid coordinates if necessary
    L_i = grid(means)
    
    # 3. Get BRDF features
    rho_i = gaussians['brdf_features']
    
    # 4. Encode View Direction
    # View direction from camera to Gaussian mean
    view_dir = means - camera.position
    view_dir = view_dir / view_dir.norm(dim=-1, keepdim=True)
    view_dir_encoded = integrated_directional_encoding(view_dir) # [G] IDE function
    
    # 5. Compute Specular Color
    c_s = renderer(rho_i, L_i, view_dir_encoded)
    
    # 6. Compute Total Radiance
    c_d = gaussians['diffuse_color']
    c = c_d + c_s
    
    # 7. Render
    rendered_image = render_gaussians(gaussians, camera, grid, renderer)
    
    # 8. Compute Loss
    loss_l1 = torch.abs(rendered_image - gt_image).mean()
    loss_dssim = 1 - ssim(rendered_image, gt_image) # [G] D-SSIM implementation
    lambda_val = 0.2
    loss = (1 - lambda_val) * loss_l1 + lambda_val * loss_dssim
    
    # 9. Backprop
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    return loss.item()
```

**Parameters:**
*   **Training Iterations:** 30,000 [S1].
*   **Diffuse-Only Phase:** First 3,000 iterations [S1].
*   **Loss Weight $\lambda$:** 0.2 [S1].
*   **Illumination Grid Resolution:** $150^3$ voxels [S1].
*   **BRDF Features:** 16x3 SH coefficients repurposed + 4 parameters (base color, roughness) [S1].

**Edge Cases and Pitfalls:**
*   **Grid Boundary:** If the scene geometry expands beyond the initial bounding box, illumination queries may be invalid. The method mitigates this by shrinking and resampling the grid midway through training to fit the Gaussians [S1].
*   **Precision:** The factorized tensor representation is designed to be memory-efficient (87.5% less voxels than TensoRF) [S1], but interpolation accuracy depends on the resolution and rank.

## Operating characteristics
*   **Time Complexity:** Rendering speed is comparable to 3DGS because illumination is pre-calculated in the grid and evaluated via fast interpolation, avoiding per-ray tracing [S1]. Training involves optimizing the grid and neural network, which adds overhead compared to vanilla 3DGS but remains efficient due to the compact tensor representation [S1].
*   **Memory Complexity:** The illumination grid uses $150^3$ voxels, which is 87.5% less memory than TensoRF on synthetic datasets [S1]. The memory footprint is low relative to the number of Gaussians, which is often an order of magnitude larger [S1].
*   **Performance:** On the Shiny Blender dataset, 3iGS achieves PSNR 27.51, SSIM 0.930, LPIPS 0.045 [S1]. It outperforms 3DGS in scenes with reflective surfaces and specular view-dependent effects [S1].
*   **Strengths:** Accurate modeling of specular highlights and view-dependent effects; fast rendering; memory-efficient illumination field.
*   **Limitations:** Requires careful tuning of grid boundaries and training schedule (diffuse-only phase); relies on local illumination assumption which may not capture global indirect lighting as effectively as global methods.

## Background
3iGS was introduced by Z.J. Tang and T.J. Cham [S1]. It addresses the limitations of 3DGS in rendering specular effects by incorporating a local illumination field and BRDF features. It builds on the principles of [Color and Appearance Modeling](14740-color-and-appearance-modeling.md) and improves upon [GaussianShader](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md) by using grid-based tensors instead of global environment maps [S1].

## Sources
[S1] 3iGS: Factorised Tensorial Illumination for 3D Gaussian Splatting, https://core.ac.uk/works/391741134, fetched 2026-10-29
