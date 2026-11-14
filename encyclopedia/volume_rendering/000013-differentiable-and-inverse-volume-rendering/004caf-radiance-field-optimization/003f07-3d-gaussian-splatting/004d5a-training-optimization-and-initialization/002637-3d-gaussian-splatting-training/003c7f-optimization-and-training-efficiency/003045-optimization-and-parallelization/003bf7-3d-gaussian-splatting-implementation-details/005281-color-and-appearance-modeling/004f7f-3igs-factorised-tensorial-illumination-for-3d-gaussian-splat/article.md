# 3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)

## Overview
3iGS enhances 3D Gaussian Splatting (3DGS) by modeling outgoing radiance as a function of a continuous local illumination field and individual Gaussian BRDF attributes, rather than optimizing outgoing radiance in isolation [S1]. It differs from its sibling [SWAG](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md) by using a factorized tensor grid for incident illumination and a neural network for specular shading, whereas SWAG conditions Gaussians on appearance embeddings [S1]. This approach significantly improves specular view-dependent effects while maintaining the rapid training and rendering speeds of the base 3DGS rasterization pipeline [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians defined by means $\boldsymbol{\mu}$, opacities $\alpha$, covariances $\boldsymbol{\Sigma}$, and initial appearance features; a scene bounding box; ground truth images $\mathbf{C}_{gt}$.
*   **Outputs:** Rendered images $\hat{\mathbf{C}}$ and optimized Gaussian attributes (means, opacities, covariances, diffuse colors, BRDF features) and illumination grid parameters.

**Data Structures:**
1.  **3D Gaussian $i$:**
    *   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Mean coordinates.
    *   $\alpha_i \in \mathbb{R}$: Opacity.
    *   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scaling matrix (diagonal).
    *   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix (orthogonal).
    *   $\mathbf{c}_{d,i} \in \mathbb{R}^3$: Diffuse color.
    *   $\boldsymbol{\rho}_i \in \mathbb{R}^{16 \times 3 + 4}$: BRDF features. The first 16x3 channels repurpose the standard 3DGS SH coefficients. The last 4 parameters encode base color and roughness for Integrated Directional Encoding (IDE) [S1].
2.  **Factorised Tensor Illumination Grid $\mathcal{G}_l$:**
    *   Represents the continuous local illumination field $L_i$ at position $\mathbf{x}_i$.
    *   Formulated using Vector-Matrix (VM) factorization [S1].
    *   Parameters: Feature tensors $\mathbf{A}_{L,r}^X, \mathbf{A}_{L,r}^Y, \mathbf{A}_{L,r}^Z$ and feature vectors $\mathbf{b}_{3r-2}, \mathbf{b}_{3r-1}, \mathbf{b}_{3r}$ for rank $r=1 \dots R_L$.
    *   Resolution: Coarse grid of $150^3$ voxels [S1].
3.  **Neural Renderer $\mathcal{F}$:**
    *   A small neural network mapping inputs to specular color [S1].

**Numbered Steps:**

1.  **Covariance Transformation:**
    Compute the anisotropic covariance matrix $\boldsymbol{\Sigma}$ from learned scaling $\mathbf{S}$ and rotation $\mathbf{R}$:
    $$ \boldsymbol{\Sigma} = \mathbf{R}\mathbf{S}\mathbf{S}^T\mathbf{R}^T $$
    Transform $\boldsymbol{\Sigma}$ to camera coordinates using viewing transformation $\mathbf{W}$ and affine approximation of projective transformation $\mathbf{J}$:
    $$ \boldsymbol{\Sigma}' = \mathbf{J}\mathbf{W}\boldsymbol{\Sigma}\mathbf{W}^T\mathbf{J}^T $$
    [S1]

2.  **Illumination Query:**
    For each Gaussian $i$ with mean $\mathbf{x}_i$, compute incoming illumination $L_i$ by interpolating from the factorised tensor illumination grid $\mathcal{G}_l$:
    $$ \mathcal{G}_l : \{ \mathbf{x}_i \} \mapsto L_i $$
    The grid value is calculated via VM factorization:
    $$ \mathcal{G}_l = \sum_{r=1}^{R_L} \mathbf{A}_{L,r}^X \circ \mathbf{b}_{3r-2} + \mathbf{A}_{L,r}^Y \circ \mathbf{b}_{3r-1} + \mathbf{A}_{L,r}^Z \circ \mathbf{b}_{3r} $$
    where $\circ$ denotes the appropriate tensor contraction/interpolation operation defined by the VM decomposition [S1].

3.  **Shading (Specular Color):**
    Encode the viewing direction $\omega_o$ using Integrated Directional Encoding (IDE). Map the Gaussian BRDF features $\boldsymbol{\rho}_i$, the queried illumination $L_i$, and the encoded view direction to the specular color $\mathbf{c}_{s,i}$ using the neural network $\mathcal{F}$:
    $$ \mathcal{F} : \{ \boldsymbol{\rho}_i, L_i, \omega_o \} \mapsto \mathbf{c}_{s,i} $$
    [S1]

4.  **Radiance Composition:**
    Compute the total outgoing radiance $\mathbf{c}_i(\omega_o)$ by linearly adding the diffuse color $\mathbf{c}_{d,i}$ and the specular color $\mathbf{c}_{s,i}$:
    $$ \mathbf{c}_i(\omega_o) = \mathbf{c}_{d,i} + \mathbf{c}_{s,i} $$
    [S1]

5.  **Rasterization:**
    Render the scene using the standard 3DGS alpha-blending rasterization pipeline. For a pixel $\mathbf{u}$, composite the colors of $N$ ordered Gaussians:
    $$ \hat{\mathbf{C}} = \sum_{i \in N} T_i g_i(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') \alpha_i \mathbf{c}_i $$
    where $T_i = \prod_{j=1}^{i-1} (1 - g_j(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') \alpha_j)$ is the transmittance, and $g_i$ is the 2D Gaussian splat function:
    $$ g(\mathbf{x} | \boldsymbol{\mu}, \boldsymbol{\Sigma}) = e^{-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})} $$
    [S1]

6.  **Optimization Training Strategy:**
    *   **Phase 1 (Iterations 0–3,000):** Train the model using only the diffuse color $\mathbf{c}_{d,i}$. Optimize means, opacity, covariance, and diffuse color. [S1]
    *   **Phase 2 (Iterations 3,000–30,000):** Add specular colors. Optimize BRDF features $\boldsymbol{\rho}_i$ and the illumination grid $\mathcal{G}_l$ alongside other parameters. [S1]
    *   **Grid Boundary Management:** Define an initial boundary encapsulating the scene bounding box. Midway through training, shrink the illumination grid to fit the Gaussians and resample the grid with the same number of voxels to maintain resolution relative to the occupied volume. [S1]

7.  **Loss Function:**
    Minimize the combined loss:
    $$ \mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda \mathcal{L}_{\text{D-SSIM}} $$
    with $\lambda = 0.2$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_covariance(S, R):
    """
    Computes the 3x3 covariance matrix from scaling S and rotation R.
    S is diagonal, R is orthogonal.
    """
    # S is typically stored as a vector of 3 scale factors
    S_mat = diag(S)
    return R @ S_mat @ S_mat.T @ R.T

def transform_covariance(Sigma, W, J):
    """
    Transforms covariance to camera coordinates.
    W: Viewing transformation matrix.
    J: Affine approximation of projective transformation.
    """
    return J @ W @ Sigma @ W.T @ J.T

def query_illumination_grid(x_i, grid_params, resolution=150):
    """
    Queries the factorised tensor illumination grid at position x_i.
    grid_params contains A_X, A_Y, A_Z and b vectors.
    Returns scalar or vector illumination L_i.
    """
    # Implementation of VM factorization interpolation
    # Sum over rank r
    L_i = 0
    for r in range(R_L):
        # Interpolate A_X, A_Y, A_Z at x_i
        ax = interpolate_tensor_1d(A_X[r], x_i.x, resolution)
        ay = interpolate_tensor_1d(A_Y[r], x_i.y, resolution)
        az = interpolate_tensor_1d(A_Z[r], x_i.z, resolution)
        
        # Combine with feature vectors b
        # Note: The formula uses specific indices for b
        term = ax * b[3*r-2] + ay * b[3*r-1] + az * b[3*r]
        L_i += term
    return L_i

def shade_gaussian(rho_i, L_i, view_dir, F_net):
    """
    Computes specular color using neural network F.
    rho_i: BRDF features (16x3 SH + 4 IDE params)
    L_i: Illumination from grid
    view_dir: View direction omega_o
    """
    # Encode view direction with IDE
    view_encoded = ide_encode(view_dir)
    
    # Concatenate features
    input_features = concatenate([rho_i, L_i, view_encoded])
    
    # Pass through small neural network
    c_s = F_net(input_features)
    return c_s

def render_gaussians(gaussians, camera_params, grid_params, F_net):
    """
    Renders the scene using 3DGS rasterization.
    """
    # 1. Transform Gaussians to camera space
    W = camera_params.W
    J = camera_params.J
    
    rendered_image = zeros_like(target_image)
    
    # Sort Gaussians by depth for alpha blending
    sorted_gaussians = sort_by_depth(gaussians, camera_params)
    
    transmittance = 1.0
    
    for g in sorted_gaussians:
        # Compute covariance in camera space
        Sigma = compute_covariance(g.S, g.R)
        Sigma_prime = transform_covariance(Sigma, W, J)
        
        # Project to 2D and compute splat
        # g_2d is the 2D Gaussian function evaluated at pixel centers
        # This part relies on standard 3DGS rasterization kernels
        
        # Query illumination
        L_i = query_illumination_grid(g.mu, grid_params)
        
        # Shade
        c_d = g.c_d
        c_s = shade_gaussian(g.rho, L_i, g.view_dir, F_net)
        c_total = c_d + c_s
        
        # Alpha blend
        weight = transmittance * g.alpha * g_2d_splat
        rendered_image += weight * c_total
        transmittance *= (1.0 - weight)
        
        if transmittance < epsilon:
            break
            
    return rendered_image

def train_step(gaussians, grid_params, F_net, gt_image, camera_params):
    """
    Single optimization step.
    """
    pred_image = render_gaussians(gaussians, camera_params, grid_params, F_net)
    
    # Loss calculation
    l1_loss = L1(pred_image, gt_image)
    ssim_loss = D_SSIM(pred_image, gt_image)
    loss = (1 - 0.2) * l1_loss + 0.2 * ssim_loss
    
    # Backpropagate to update:
    # Gaussians: mu, alpha, S, R, c_d, rho
    # Grid: A_X, A_Y, A_Z, b
    # Network: F_net weights
    
    return loss
```

**Edge Cases and Pitfalls:**
*   **Grid Boundary:** If the illumination grid boundary is not shrunk midway through training, it may cover empty space, wasting memory and potentially providing inaccurate illumination for Gaussians near the edges. The sources specify shrinking the grid to fit the Gaussians and resampling with the same number of voxels [S1].
*   **Training Stability:** Specular effects are unstable if introduced immediately. The sources mandate training with only diffuse color for the first 3,000 iterations before adding specular components [S1].
*   **BRDF Feature Initialization:** The BRDF features repurpose the 16x3 SH coefficients. These must be initialized appropriately from the base 3DGS model [S1].

## Operating characteristics
*   **Time Complexity:** Rendering speed is comparable to 3DGS because the illumination field is pre-calculated and stored in volumes, allowing fast evaluation via interpolation [S1]. The neural network $\mathcal{F}$ is small, adding minimal overhead per Gaussian [S1].
*   **Memory Complexity:** The illumination field uses a coarse resolution of $150^3$ voxels, which is 87.5% less memory than TensoRF on synthetic datasets [S1]. The memory footprint is low compared to the number of optimized Gaussians, which is often an order of magnitude higher [S1].
*   **Performance:** On the Shiny Blender dataset, 3iGS achieves an average PSNR of 27.51, SSIM of 0.930, and LPIPS of 0.045 [S1]. It surpasses 3DGS quantitatively and qualitatively on synthetic datasets (NeRF Blender) and real-world scenarios (Tanks and Temples) [S1].
*   **Strengths:** Significantly enhances specular view-dependent effects; maintains rapid training and rendering speeds; compact illumination representation [S1].
*   **Limitations:** Relies on a coarse grid which may miss fine-scale lighting details compared to ray-traced methods; requires careful tuning of the training schedule (diffuse-only phase) [S1].
*   **When to use:** Scenes with reflective surfaces where colors change dramatically across viewing angles (e.g., glossy materials) [S1].
*   **When not to use:** Scenes where global environment lighting is critical and cannot be approximated by a local grid, or where extreme geometric detail in lighting is required beyond the grid resolution [S1].

## Background
3iGS was introduced by Z.J. Tang and T.J. Cham [S1]. It addresses the limitations of standard 3DGS, which optimizes outgoing radiance in isolation without accounting for adjacent Gaussians or scene lighting conditions, resulting in poor view-dependent effects for specular surfaces [S1]. The method is inspired by conventional computer graphics engines that use baking techniques for fast rendering [S1]. It differs from GaussianShader [13] by using a local grid-based tensor illumination field instead of a global differentiable environment light stored in cube maps, and by optimizing Gaussian BRDF features relative to this field rather than using independent SH-based color [S1].

## Sources
[S1] 3iGS: Factorised Tensorial Illumination for 3D Gaussian Splatting, https://core.ac.uk/works/391741134, fetched 2026-10-29
