# Negative Gaussian Splatting

Also called: NegGS, Diff-Gaussian

## Overview
Negative Gaussian Splatting (NegGS) extends the 3D Gaussian Splatting framework by incorporating "negative Gaussians"—ellipsoids with negative color values—into the scene representation [S1, S2]. This allows the modeling of high-frequency elements, complex nonlinear structures, and rapid color transitions that standard positive Gaussians represent inefficiently [S1, S2]. Unlike standard Gaussian Splatting, which requires many components to approximate complex shapes, NegGS uses the cancellation effect between positive and negative Gaussians to achieve greater flexibility in shape and color modulation with fewer total primitives [S1, S2].

## How it works
NegGS operates within the differentiable rendering framework of 3D Gaussian Splatting [Parent](10145-representation-and-kernel-modifications.md). It modifies the primitive representation and optimization strategy to support negative color coefficients.

### Inputs and Outputs
*   **Inputs:** A set of camera poses and corresponding 2D images; an initial point cloud or grid for initialization.
*   **Outputs:** A set of 3D Gaussians $G_{NeGS}$ optimized to minimize the reconstruction loss between rendered and ground truth images.

### Data Structures
The scene is represented by a joint family of Gaussians $G_{NeGS}$, partitioned into two subsets:
1.  $G_{GS}$: Tuples with positive color values $c_i$.
2.  $G_{-GS}$: Tuples with negative color values $c_i$ [S1].

Each Gaussian $i$ is defined by the standard 3DGS parameters:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$
*   Covariance $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (parameterized by rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$)
*   Opacity $\alpha_i \in [0, 1]$
*   Color $\mathbf{c}_i \in \mathbb{R}^3$ (or spherical harmonics coefficients). Crucially, $\mathbf{c}_i$ can be negative [S1].

To preserve rendering speed and leverage the original GS implementation, negative Gaussians are tracked via a binary mask [S1]. The rendering pipeline remains identical to standard GS, but the alpha blending accumulation uses the signed color values [S1].

### The Diff-Gaussian Concept
The method is motivated by the "Diff-Gaussian" distribution, defined as the ratio of two Gaussian probability density functions (PDFs) [S2]:
$$ f_{c_{opt}}(w) = \frac{f_{m_0, \Sigma_0}(w)}{f_{m_1, \Sigma_1}(w)} $$
This distribution can approximate complex nonlinear structures (e.g., donut or moon-shaped datasets) that single Gaussians cannot [S2].

**Optimal Coefficient Derivation:**
The optimal coefficient $c_{opt}$ for a Diff-Gaussian formed by $f_{m_0, \Sigma_0}$ and $f_{m_1, \Sigma_1}$ is the minimum of the ratio [S1]:
$$ c_{opt} := \min_x \frac{f_{m_0, \Sigma_0}(x)}{f_{m_1, \Sigma_1}(x)} = \frac{f_{m_0, \Sigma_0}(w)}{f_{m_1, \Sigma_1}(w)} $$
This minimum is achieved if and only if the vector $w$ satisfies [S1]:
$$ \Sigma_1^{-1}(w - m_1) = \Sigma_0^{-1}(w - m_0) $$
This condition is derived by minimizing the quadratic function $q(x) = x^T A x + 2b^T x$, where [S1]:
$$ A = \Sigma_1^{-1} - \Sigma_0^{-1} \quad \text{and} \quad b = \Sigma_0^{-1} m_0 - \Sigma_1^{-1} m_1 $$
Assuming $A$ is positive semi-definite, $w$ is the solution to the linear system above [S1].

**Null Density Vector for Diagonal Covariances:**
For diagonal covariance matrices $\Sigma_0 = \text{diag}(\sigma_1^2, \dots, \sigma_n^2)$ and $\Sigma_1 = \text{diag}(h_1 \sigma_1^2, \dots, h_n \sigma_n^2)$ with $0 < h_i \le 1$, the coordinates $w_i$ of the null density vector (where the ratio is minimized) are [S1]:
1.  $w_i = \frac{m_{1,i} - h_i m_{0,i}}{1 - h_i}$ if $h_i < 1$
2.  $w_i \in \mathbb{R}$ if $h_i = 1$ and $m_{1,i} = m_{0,i}$

### Algorithm Steps
1.  **Initialization:** Initialize the scene with a set of Gaussians. The initial number of negative Gaussians is a critical hyperparameter, typically set to approximately 20% of the total Gaussians [S1].
2.  **Differentiable Rasterization:** Project 3D Gaussians to 2D ellipses. Sort by depth. Accumulate color using alpha blending, allowing negative color values [S1].
3.  **Loss Computation:** Compute the reconstruction loss (e.g., L1, SSIM) between the rendered image and the ground truth [Parent](10145-representation-and-kernel-modifications.md).
4.  **Gradient-Based Optimization:** Update parameters ($\mathbf{p}_i, \boldsymbol{\Sigma}_i, \alpha_i, \mathbf{c}_i$) using gradient descent [Parent](10145-representation-and-kernel-modifications.md).
5.  **Dynamic Adaptation:**
    *   **Pruning:** Gaussians with low opacity are reduced/removed [S1].
    *   **Densification:** New Gaussians are introduced into empty 3D regions to adapt the number of components [S1].
    *   Note: The balance between positive and negative Gaussians is maintained as hyperparameters, adapted via grid search during cross-validation [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# NegGS Rendering and Optimization Loop
# Assumes standard 3DGS rasterization kernel is available but modified to support signed colors

def initialize_neggs(num_total_gaussians, negative_ratio=0.2):
    """
    Initialize the joint family G_NeGS = G_GS U G_-GS.
    """
    num_negative = int(num_total_gaussians * negative_ratio)
    num_positive = num_total_gaussians - num_negative
    
    # Initialize positions, covariances, opacities from SfM or grid
    p = initialize_positions(num_total_gaussians)
    sigma = initialize_covariances(num_total_gaussians)
    alpha = initialize_opacities(num_total_gaussians)
    
    # Initialize colors. Positive Gaussians have positive colors.
    # Negative Gaussians have negative colors.
    c = initialize_colors(num_total_gaussians)
    # Ensure negative Gaussians have negative color values
    for i in range(num_negative):
        c[i] = -abs(c[i])
        
    # Binary mask to track negative Gaussians for potential specialized handling
    is_negative = torch.zeros(num_total_gaussians, dtype=torch.bool)
    is_negative[:num_negative] = True
    
    return {
        'p': p, 'sigma': sigma, 'alpha': alpha, 'c': c, 'is_negative': is_negative
    }

def render_neggs(gaussians, camera_params):
    """
    Render image using NegGS primitives.
    Identical to standard GS rasterization, but c_i can be negative.
    """
    # 1. Project 3D Gaussians to 2D
    # 2. Sort by depth
    # 3. Alpha blend
    # Color accumulation: C = sum(T_i * alpha_i * c_i)
    # where c_i is signed.
    image = standard_gs_rasterize(gaussians['p'], gaussians['sigma'], 
                                  gaussians['alpha'], gaussians['c'], camera_params)
    return image

def optimize_neggs(gaussians, images, camera_params, num_iterations):
    """
    Optimize NegGS parameters.
    """
    for iteration in range(num_iterations):
        # Sample a random camera
        camera = sample_camera(camera_params)
        
        # Render
        rendered_image = render_neggs(gaussians, camera)
        ground_truth = get_ground_truth(camera)
        
        # Compute Loss (L1 + SSIM)
        loss = compute_loss(rendered_image, ground_truth)
        
        # Backpropagate
        loss.backward()
        
        # Update parameters
        update_parameters(gaussians)
        
        # Dynamic Adaptation
        # Prune low opacity Gaussians
        gaussians = prune_low_opacity(gaussians, threshold=0.005)
        
        # Densify in empty regions / high gradient areas
        gaussians = densify_gaussians(gaussians, camera, threshold=0.01)
        
        # Note: The ratio of negative Gaussians is a hyperparameter.
        # If the count of negative Gaussians deviates significantly from target,
        # re-initialize or adjust via grid-search derived strategy [S1].
        
    return gaussians
```

### Parameters and Edge Cases
*   **Initial Negative Ratio:** Approximately 20% of total Gaussians [S1].
*   **Covariance Constraint:** In the theoretical derivation of Diff-Gaussians, $\Sigma_0 - \Sigma_1$ is assumed to be positive semi-definite [S1].
*   **Precision:** The method relies on the cancellation of positive and negative values. Numerical stability during alpha blending with negative colors must be handled carefully to avoid artifacts, though the standard GS implementation is leveraged [S1].
*   **Degenerate Input:** If $h_i = 1$ and $m_{1,i} = m_{0,i}$ in the diagonal covariance case, the null density vector coordinate $w_i$ is unconstrained ($\mathbb{R}$) [S1].

## Operating characteristics
*   **Time Complexity:** NegGS alleviates the time complexity increase associated with using many Gaussian components to model complex nonlinear structures [S2]. It fully leverages the original GS framework implementation, preserving the original GS rendering speed [S1].
*   **Memory Complexity:** Requires storage for negative color values and a binary mask to track negative Gaussians [S1].
*   **Performance:**
    *   On Mip-NeRF360, NegGS-30K achieved an average PSNR of 27.39, SSIM of 0.812, and LPIPS of 0.219, compared to GS-30K's 27.21 PSNR, 0.815 SSIM, and 0.214 LPIPS [S1].
    *   On NeRF Synthetic, NegGS achieved slightly higher PSNR (e.g., 35.88 vs 35.82 for Chair) and comparable SSIM/LPIPS [S1].
*   **Strengths:** Better modeling of high-frequency elements with rapid color transitions; improved representation of shadows; ability to approximate complex nonlinear structures with fewer primitives [S1, S2].
*   **Limitations:** Requires careful initialization of the negative Gaussian ratio; optimization efficacy is sensitive to the initial number of negative Gaussians [S1].
*   **When to Use:** Scenes with complex nonlinear geometries, high-frequency details, or rapid color transitions where standard GS requires excessive primitives [S1, S2].
*   **When Not to Use:** Simple scenes where standard GS is sufficient, or when computational resources for grid-search hyperparameter tuning are limited [S1].

## Background
Negative Gaussian Splatting (NegGS) is the first paper to extend the simple ellipsoid shapes of Gaussian Splatting to more complex nonlinear structures [S2]. It was introduced to address the limitation that standard Gaussians are restrictive due to their perceived linear nature, often requiring a multitude of components to reflect complex, nonlinear structures accurately, which increases time complexity [S2]. The method was evaluated on NeRF Synthetic, Mip-NeRF360, Tanks and Temples, Deep Blending, and Shiny Blender datasets [S1].

## Sources
[S1] NegGS: Negative Gaussian Splatting, https://doi.org/10.1016/j.ins.2025.121912, fetched 2026-10-26
[S2] NegGS: Negative Gaussian Splatting, https://core.ac.uk/download/630026911.pdf, fetched 2026-10-28
