# NeRF-In Inpainting Optimization

Also called: Free-Form NeRF Inpainting with RGB-D Priors, NeRF object removal, NeRF-In

## Overview
NeRF-In is a framework for removing unwanted objects from a pre-trained Neural Radiance Field (NeRF) by optimizing its parameters $\Theta$ based on user-provided masks and RGB-D priors, without requiring category-specific training data [S1, unofficial]. It differs from sibling methods like [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md) and [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md) by explicitly combining color guidance from 2D inpainting models with depth guidance from depth completion methods to ensure both visual plausibility and geometric consistency across multiple views [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Pre-trained NeRF model $F_\Theta$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.
    *   User-chosen camera position $\mathbf{o}_u$.
    *   User-drawn free-form mask $M_u$ on the rendered view $I_u = F_{\Theta}^{image}(\mathbf{o}_u)$, where $M_u(\mathbf{p}) = 1$ for pixels outside the masked region (background/valid) and $0$ inside (object to remove) [S1, unofficial].
*   **Outputs:**
    *   Updated NeRF parameters $\tilde{\Theta}$ that remove the unwanted object in the masked region across all views [S1, unofficial].

### Data Structures
*   **NeRF Model $F_\Theta$:** Standard MLP-based NeRF representation.
    *   Coarse model: Uses $N_c = 64$ sample points per ray [S1, unofficial].
    *   Fine model: Uses $N_f = 128$ sample points per ray [S1, unofficial].
*   **Masks:**
    *   $M_u$: Binary mask on user view.
    *   $M = \{M_s \mid s = 1 \dots K\}$: Transferred masks for sampled views, generated via video object segmentation [S1, unofficial].
*   **Guidance Materials:**
    *   $I^G = \{I_s^G \mid s = 1 \dots K\}$: Guiding color images.
    *   $D^G = \{D_s^G \mid s = 1 \dots K\}$: Guiding depth images.
    *   $I_u^G$: Guiding color image for the user view.
    *   $D_u^G$: Guiding depth image for the user view.

### Numbered Steps

1.  **Initialization:** Initialize NeRF parameters $\Theta$ (typically using a pre-trained model) [S1, unofficial].
2.  **User Input:** User renders view $I_u$ from $F_\Theta$ at $\mathbf{o}_u$ and draws mask $M_u$ [S1, unofficial].
3.  **View Sampling:** Sample $K$ camera positions $\mathcal{O} = \{\mathbf{o}_s \mid s = 1 \dots K\}$ along a pre-set trajectory (e.g., LLFF test-set trajectory) [S1, unofficial].
4.  **Initial Rendering:** Render initial RGB images $I_s$ and depth images $D_s$ for all sampled views $\mathbf{o}_s$ using $F_\Theta$ [S1, unofficial].
5.  **Mask Transfer:** Transfer $M_u$ to sampled views to generate $M = \{M_s\}$ using a video object segmentation method (e.g., STCN) [S1, unofficial].
6.  **Guidance Generation:**
    *   Generate guiding color images $I_s^G = \rho(I_s, M_s)$ using an image inpainting method $\rho$ (e.g., MST inpainting network) [S1, unofficial].
    *   Generate guiding depth images $D_s^G = \tau(D_s, M_s, I_s^G)$ using a depth completion method $\tau$ (e.g., Fast Bilateral Solver) [S1, unofficial].
    *   Note: The framework sets the set of all views $\mathcal{O}_{all} = \{\mathbf{o}_u\}$ and outside views $\mathcal{O}_{out} = \mathcal{O}$ to maintain multi-view consistency [S1, unofficial].
7.  **Optimization:** Optimize $\Theta$ to $\tilde{\Theta}$ by minimizing the joint loss $L_{color}(\Theta) + L_{depth}(\Theta)$ using the Adam optimizer for 50,000 steps [S1, unofficial].

### Formulas

**Volume Rendering:**
The expected color $\hat{C}(\mathbf{r})$ of a camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ is computed via numerical quadrature [S1, unofficial]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T(t_i) (1 - \exp(-\sigma(t_i)\delta_i)) c(t_i)
$$

where:
*   $N$ is the number of quadrature points (64 for coarse, 128 for fine).
*   $t_i$ are sample points between near plane $t_n$ and far plane $t_f$.
*   $\delta_i = t_{i+1} - t_i$.
*   $c(t_i)$ and $\sigma(t_i)$ are color and density predicted by $F_\Theta$ at $t_i$.
*   $T(t_i) = \exp\left(-\sum_{j=1}^{i-1} \sigma(t_j)\delta_j\right)$ is the accumulated transmittance.

**Depth Rendering:**
Rendered depth $D(\mathbf{o}_s)$ is computed by accumulating density $\sigma$ from ray batches during volume rendering [S1, unofficial]. Specifically, $D_f(\mathbf{o}_s)$ and $D_c(\mathbf{o}_s)$ denote fine and coarse volume predicted depth images, respectively.

**Color-Guiding Loss:**
The total color loss is the sum of all-view and outside-view losses [S1, unofficial]:

$$
L_{color}(\Theta) = L_{all}^{color}(\Theta) + L_{out}^{color}(\Theta)
$$

Where:
*   $L_{all}^{color}(\Theta)$ measures color difference on the user-chosen view $\mathbf{o}_u$:
    $$
    L_{all}^{color}(\Theta) = \| F_{\Theta}^{image}(\mathbf{o}_u) - I_u^G \|_2^2
    $$
*   $L_{out}^{color}(\Theta)$ measures color difference outside the mask on sampled views $\mathcal{O}$:
    $$
    L_{out}^{color}(\Theta) = \sum_{\mathbf{o}_s \in \mathcal{O}} \| (F_{\Theta}^{image}(\mathbf{o}_s) - I_s^G) \odot M_s \|_2^2
    $$
    *(Note: The source defines $L_{out}^{color}$ with mask $M_o$ in Eq. 10, but context implies $M_s$ for view $s$. The mask $M_s$ is 1 outside the object, so this term penalizes changes to the background.)*

**Depth-Guiding Loss:**
Measures L2 distance between rendered depth and guiding depth for sampled views [S1, unofficial]:

$$
L_{depth}(\Theta) = \sum_{\mathbf{o}_s \in \mathcal{O}} \left( \| D_f(\mathbf{o}_s) - D_s^G \|_2^2 + \| D_c(\mathbf{o}_s) - D_s^G \|_2^2 \right)
$$

**Total Optimization Objective:**
$$
\tilde{\Theta} := \arg \min_{\Theta} \left( L_{color}(\Theta) + L_{depth}(\Theta) \right)
$$

### Parameters
*   **K:** Number of sampled camera positions. Not given in the sources.
*   **Batch size:** 4,096 [S1, unofficial].
*   **Optimizer:** Adam [S1, unofficial].
*   **Training steps:** 50,000 steps for inpainting optimization [S1, unofficial].
*   **Hardware:** Intel i7-7800X CPU and GTX-1080 GPU [S1, unofficial].

### Edge Cases and Pitfalls
*   **Masked MSE only:** Optimizing only with masked Mean Squared Error (MSE) on transferred masks without explicit guidance causes the unwanted object to remain in the result [S1, unofficial].
*   **Depth-only loss:** Using only $L_{depth}$ introduces color noise outside the masked region and loses color information in flat regions [S1, unofficial].
*   **Multiple guidance views:** Using multiple inpainted images as color guidance (beyond the user view) introduces visual inconsistency between views [S1, unofficial].
*   **Guidance Artifacts:** If the initial guidance materials (from $\rho$ and $\tau$) contain artifacts, the optimized NeRF may output undesired results [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def volume_render_color(F_theta, ray_origins, ray_directions, N_samples, near, far):
    """
    Renders color using volume rendering integral.
    F_theta: NeRF model (MLP)
    ray_origins: (B, 3)
    ray_directions: (B, 3)
    N_samples: int (64 or 128)
    """
    # Sample points along rays
    t_vals = torch.linspace(0., 1., N_samples, device=ray_origins.device)
    t_vals = near + (far - near) * t_vals
    t_vals = t_vals.view(1, -1).expand(ray_origins.shape[0], -1)
    
    # Add noise for regularization (standard NeRF practice, not explicitly in S1 but implied by 'numerical quadrature' and standard implementation)
    # [G] Standard NeRF adds uniform noise to t_vals to break symmetry.
    noise = torch.rand_like(t_vals)
    t_vals = t_vals + (1.0 / N_samples) * noise
    
    # Interpolate points
    pts = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_vals.unsqueeze(2)
    
    # Query NeRF for density and color
    # F_theta returns (sigma, color)
    # [G] Assuming F_theta takes (pts, dirs) and returns (sigma, c)
    # This is a simplification; actual NeRF uses positional encoding.
    # We assume F_theta is a callable that handles encoding internally.
    sigma, c = F_theta(pts, ray_directions)
    
    # Compute transmittance
    deltas = t_vals[:, 1:] - t_vals[:, :-1]
    # Add infinite delta for last point
    deltas = torch.cat([deltas, torch.tensor([1e10], device=pts.device).expand(deltas.shape[0], 1)], dim=1)
    
    alpha = 1.0 - torch.exp(-sigma * deltas)
    T = torch.cumprod(1.0 - alpha + 1e-10, dim=1)[:, :-1]
    weights = alpha * T
    
    # Composite color
    C = torch.sum(weights.unsqueeze(-1) * c, dim=1)
    return C

def render_depth(F_theta, ray_origins, ray_directions, N_samples, near, far):
    """
    Renders depth by accumulating density-weighted distances.
    """
    # Re-use sampling logic from volume_render_color
    t_vals = torch.linspace(0., 1., N_samples, device=ray_origins.device)
    t_vals = near + (far - near) * t_vals
    t_vals = t_vals.view(1, -1).expand(ray_origins.shape[0], -1)
    noise = torch.rand_like(t_vals)
    t_vals = t_vals + (1.0 / N_samples) * noise
    
    pts = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_vals.unsqueeze(2)
    sigma, _ = F_theta(pts, ray_directions)
    
    deltas = t_vals[:, 1:] - t_vals[:, :-1]
    deltas = torch.cat([deltas, torch.tensor([1e10], device=pts.device).expand(deltas.shape[0], 1)], dim=1)
    
    alpha = 1.0 - torch.exp(-sigma * deltas)
    T = torch.cumprod(1.0 - alpha + 1e-10, dim=1)[:, :-1]
    weights = alpha * T
    
    # Depth is weighted average of t_vals
    D = torch.sum(weights * t_vals, dim=1)
    return D

def nerf_in_optimization(F_theta, o_u, M_u, K, trajectory, rho, tau, num_steps=50000):
    """
    NeRF-In Inpainting Optimization Loop.
    F_theta: Pre-trained NeRF model.
    o_u: User-chosen camera position.
    M_u: User-drawn mask (1 for background, 0 for object).
    K: Number of sampled views.
    trajectory: Function or list of camera positions.
    rho: Image inpainting function.
    tau: Depth completion function.
    """
    optimizer = optim.Adam(F_theta.parameters(), lr=1e-5) # [G] Typical LR for NeRF, not specified in S1
    
    # 1. Sample K camera positions
    sampled_views = trajectory(K) # Returns list of K camera positions
    
    # 2. Render initial views to get initial images and depths for guidance generation
    # [G] Assuming helper functions to render full images from NeRF given camera params
    initial_images = [render_image(F_theta, o) for o in sampled_views]
    initial_depths = [render_depth_image(F_theta, o) for o in sampled_views]
    
    # 3. Transfer mask to sampled views
    # M_s = STCN(M_u, I_u, I_s)
    transferred_masks = [transfer_mask(M_u, initial_images[i], sampled_views[i]) for i in range(K)]
    
    # 4. Generate Guidance Materials
    # I_s^G = rho(I_s, M_s)
    guiding_colors = [rho(initial_images[i], transferred_masks[i]) for i in range(K)]
    
    # D_s^G = tau(D_s, M_s, I_s^G)
    guiding_depths = [tau(initial_depths[i], transferred_masks[i], guiding_colors[i]) for i in range(K)]
    
    # 5. Optimization Loop
    for step in range(num_steps):
        optimizer.zero_grad()
        
        # Define sets of views
        # O_all = {o_u}, O_out = {o_s | s=1..K}
        
        # --- Color Loss ---
        
        # L_all_color: User view
        # Render user view
        I_u_rendered = render_image(F_theta, o_u)
        # [G] Assuming I_u^G is generated similarly to I_s^G for user view
        # I_u^G = rho(I_u, M_u)
        I_u_guidance = rho(render_image(F_theta, o_u), M_u)
        
        loss_all_color = torch.mean((I_u_rendered - I_u_guidance) ** 2)
        
        # L_out_color: Sampled views, outside mask
        loss_out_color = 0.0
        for s in range(K):
            I_s_rendered = render_image(F_theta, sampled_views[s])
            I_s_guidance = guiding_colors[s]
            M_s = transferred_masks[s]
            
            # Mask is 1 outside object. We want to preserve background.
            # Loss is applied to background pixels.
            diff = (I_s_rendered - I_s_guidance) * M_s
            loss_out_color += torch.mean(diff ** 2)
            
        loss_color = loss_all_color + loss_out_color
        
        # --- Depth Loss ---
        # L_depth: Sampled views, fine and coarse depth
        loss_depth = 0.0
        for s in range(K):
            # Render fine and coarse depth
            # [G] NeRF-In uses coarse and fine models. 
            # Assuming F_theta has access to both or we render twice.
            D_f = render_depth_image(F_theta, sampled_views[s], N_samples=128)
            D_c = render_depth_image(F_theta, sampled_views[s], N_samples=64)
            D_g = guiding_depths[s]
            
            loss_depth += torch.mean((D_f - D_g) ** 2)
            loss_depth += torch.mean((D_c - D_g) ** 2)
            
        # --- Total Loss ---
        total_loss = loss_color + loss_depth
        total_loss.backward()
        optimizer.step()
        
    return F_theta
```

## Operating characteristics

*   **Time Complexity:** The optimization takes approximately 5 hours per scene on an Intel i7-7800X CPU and GTX-1080 GPU for 50,000 steps [S1, unofficial].
*   **Memory Complexity:** Depends on the NeRF model size and batch size (4,096 rays) [S1, unofficial].
*   **Strengths:**
    *   Does not require category-specific training data or procedures [S1, unofficial].
    *   Ensures geometric consistency via depth guidance [S1, unofficial].
    *   Maintains multi-view consistency by limiting color guidance to the user view and using depth guidance for other views [S1, unofficial].
*   **Limitations:**
    *   Relies on the quality of external 2D inpainting and depth completion models [S1, unofficial].
    *   May produce artifacts if guidance materials are flawed [S1, unofficial].
    *   Baseline methods (per-view color updating or masked MSE retraining) fail to produce correct depth or visual consistency [S1, unofficial].
*   **When to use:** When removing unwanted objects from a pre-trained NeRF without access to category-specific data or the ability to retrain the entire NeRF from scratch with new data.
*   **When not to use:** When high-quality depth priors are unavailable or when the object to be removed is complex and requires semantic understanding beyond geometric masking.

## Background

*   **Origin:** NeRF-In was introduced by Hao-Kang Liu, I-Chao Shen, and Bing-Yu Chen in the paper "NeRF-In: Free-Form NeRF Inpainting with RGB-D Priors" (arXiv:2206.04901, June 10, 2022) [S1, unofficial].
*   **History:** It is the first framework to enable free-form object removal from a pre-trained NeRF without category-specific training. It builds on the principles of Neural Radiance Fields [23] and image inpainting techniques [1, 9, 33].
*   **Usage:** Evaluated on the LLFF dataset and a custom dataset with 24 camera positions per scene [S1, unofficial]. Implemented in PyTorch and Python 3.9 [S1, unofficial].

## Sources

[S1] NeRF-In: Free-Form NeRF Inpainting with RGB-D Priors, https://doi.org/10.48550/arxiv.2206.04901, fetched 2026-10-26 (unofficial)
