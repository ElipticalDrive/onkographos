# NeRF-In Inpainting Optimization

Also called: Free-Form NeRF Inpainting with RGB-D Priors, NeRF object removal, NeRF-In

## Overview
NeRF-In is a framework for removing unwanted objects from a pre-trained Neural Radiance Field (NeRF) by optimizing the model parameters $\Theta$ using color and depth guidance derived from 2D images [S1, unofficial]. It enables free-form inpainting without category-specific training data or procedures [S1, unofficial]. Unlike sibling methods that may require semantic segmentation networks or feature distillation [12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md, 12491-nerf-editing-procedure.md], NeRF-In relies on explicit 2D priors (inpainting and depth completion) transferred across multiple views to guide the optimization [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
1.  A pre-trained NeRF model $F_\Theta$ with parameters $\Theta$.
2.  A user-drawn binary mask $M_u$ on a user-chosen rendered view $I_u = F_{\Theta_{image}}(o_u)$, where $M_u=1$ for pixels outside the masked region (background/valid) and $M_u=0$ inside (object to remove) [S1, unofficial].
3.  A pre-set camera trajectory (e.g., LLFF test-set trajectory) [S1, unofficial].

**Outputs:**
1.  Updated NeRF parameters $\tilde{\Theta}$ that remove the unwanted object in the masked region across all views [S1, unofficial].

### Data Structures
*   **NeRF Model $F_\Theta$**: A multi-layer perceptron (MLP) taking a 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ as input, outputting density $\sigma$ and color $\mathbf{c}$. The specific architecture (layers, dimensions) is not detailed in the source, but it follows the standard NeRF formulation [S1, unofficial].
*   **Masks**:
    *   $M_u$: User-drawn mask on view $o_u$.
    *   $M = \{M_s \mid s=1...K\}$: Transferred masks for sampled views $o_s$, generated via video object segmentation [S1, unofficial].
*   **Guiding Materials**:
    *   $I^G = \{I_s^G \mid s=1...K\}$: Guiding color images.
    *   $D^G = \{D_s^G \mid s=1...K\}$: Guiding depth images.
    *   $I_u^G$: Guiding color image for the user-chosen view.
    *   $D_u^G$: Guiding depth image for the user-chosen view.

### Numbered Steps

1.  **Initialize**: Initialize NeRF parameters $\Theta$ (using a pre-trained model for the main method) [S1, unofficial].
2.  **Sample Views**: Sample $K$ camera positions $O = \{o_s \mid s=1...K\}$ along a pre-set trajectory [S1, unofficial].
3.  **Render Initial Views**: Render RGB images $I_s$ and depth images $D_s$ from $F_\Theta$ at positions $o_s$ [S1, unofficial].
4.  **Transfer Mask**: Transfer the user-drawn mask $M_u$ to the sampled views to generate $M = \{M_s\}$ using a video object segmentation method (e.g., STCN) [S1, unofficial].
5.  **Generate Guiding Materials**:
    *   Generate guiding color images $I_s^G$ using an image inpainting method $\rho$: $I_s^G = \rho(I_s, M_s)$ [S1, unofficial].
    *   Generate guiding depth images $D_s^G$ using a depth completion method $\tau$: $D_s^G = \tau(D_s, M_s, I_s^G)$ [S1, unofficial].
    *   Note: The framework sets $O_{all} = \{o_u\}$ and $O_{out} = O$ (all sampled views) for the loss computation [S1, unofficial].
6.  **Optimize Parameters**: Optimize $\Theta$ to obtain $\tilde{\Theta}$ by minimizing the joint inpainting formulation $\Phi(M_u, I_u^G, M, D_s^G)$, which consists of a color-guiding loss $L_{color}(\Theta)$ and a depth-guiding loss $L_{depth}(\Theta)$ [S1, unofficial].

### Formulas

**Volume Rendering**
The expected color $\hat{C}(\mathbf{r})$ of a camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ is computed via numerical quadrature [S1, unofficial]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T(t_i) (1 - \exp(-\sigma(t_i)\delta_i)) c(t_i)
$$

where:
*   $N$ is the number of quadrature points sampled between near plane $t_n$ and far plane $t_f$.
*   $\delta_i = t_{i+1} - t_i$ is the distance between adjacent points.
*   $c(t_i)$ and $\sigma(t_i)$ are the color and density at point $t_i$ produced by $F_\Theta$.
*   $T(t_i) = \exp\left(-\sum_{j=1}^{i-1} \sigma(t_j)\delta_j\right)$ is the accumulated transmittance.

**Depth Rendering**
The rendered depth $D_f(\mathbf{o}_s)$ is computed by accumulating density $\sigma$ from ray batches during volume rendering [S1, unofficial]. Both coarse ($D_c$) and fine ($D_f$) depth images are used in the loss [S1, unofficial].

**Color-Guiding Loss**
The total color-guiding loss is the sum of an all-view loss and an outside-view loss [S1, unofficial]:

$$
L_{color}(\Theta) = L_{all}^{color}(\Theta) + L_{out}^{color}(\Theta)
$$

Defined as:
$$
L_{all}^{color}(\Theta) = \sum_{\mathbf{o} \in O_{all}} \| F_{\Theta_{image}}(\mathbf{o}) - I_{\mathbf{o}}^G \|_2^2
$$
$$
L_{out}^{color}(\Theta) = \sum_{\mathbf{o} \in O_{out}} \| (F_{\Theta_{image}}(\mathbf{o}) - I_{\mathbf{o}}^G) \odot M_{\mathbf{o}} \|_2^2
$$

Where:
*   $O_{all} = \{o_u\}$ (user-chosen view).
*   $O_{out} = O$ (sampled views).
*   $M_{\mathbf{o}}$ is the transferred mask for view $\mathbf{o}$.
*   The source notes that using multiple inpainted images as color guidance introduces visual inconsistency, so $L_{all}^{color}$ is restricted to the user-chosen view [S1, unofficial].

**Depth-Guiding Loss**
The depth-guiding loss measures the L2 distance between rendered fine/coarse depth images and guiding depth [S1, unofficial]:

$$
L_{depth}(\Theta) = \sum_{\mathbf{o}_s \in O} \left( \| D_f(\mathbf{o}_s) - D_{\mathbf{o}_s}^G \|_2^2 + \| D_c(\mathbf{o}_s) - D_{\mathbf{o}_s}^G \|_2^2 \right)
$$

**Total Loss**
$$
\tilde{\Theta} := \arg \min_{\Theta} L_{color}(\Theta) + L_{depth}(\Theta)
$$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def volume_render_color(F_theta, ray_origins, ray_directions, N_samples, N_coarse, N_fine):
    """
    Renders color using the standard NeRF volume rendering integral.
    F_theta: NeRF MLP model.
    ray_origins: (B, 3)
    ray_directions: (B, 3)
    N_samples: Total number of samples per ray.
    Returns: (B, 3) color, (B, 1) transmittance, (B, N_samples) weights, (B, N_samples) depths
    """
    # 1. Sample points along rays
    t_vals = torch.linspace(0., 1., N_samples).to(ray_origins.device)
    z_vals = 0.1 * t_vals + 0.9 * t_vals ** 2  # Example sampling strategy, not specified in source
    # Note: Source does not specify exact sampling strategy (e.g., uniform vs importance).
    # Assuming uniform sampling for simplicity as per standard NeRF [23] referenced in S1.
    
    # Expand dimensions for ray batching
    # ... (standard NeRF sampling logic) ...
    
    # 2. Query NeRF for sigma and color
    # pts = ray_origins[:, None, :] + ray_directions[:, None, :] * z_vals[:, :, None]
    # sigma, color = F_theta(pts, ray_directions[:, None, :].expand_as(pts))
    
    # 3. Compute transmittance and accumulated color
    # T = torch.cumprod(torch.exp(-sigma * dt), dim=1)
    # C = torch.sum(weights * color, dim=1)
    
    # Placeholder for actual implementation details not fully specified in S1
    raise NotImplementedError("Standard NeRF volume rendering implementation required")

def compute_depth(F_theta, ray_origins, ray_directions, N_samples):
    """
    Computes depth by accumulating density.
    """
    # Placeholder for depth computation logic
    raise NotImplementedError("Depth computation logic required")

def inpaint_color(I, M, inpainter):
    """
    Generates guiding color image using inpainting method rho.
    I: Input image.
    M: Mask (0 for hole, 1 for valid).
    inpainter: Function rho (e.g., MST inpainting network).
    """
    return inpainter(I, M)

def complete_depth(D, M, I_G, depth_completer):
    """
    Generates guiding depth image using depth completion method tau.
    D: Input depth.
    M: Mask.
    I_G: Guiding color image.
    depth_completer: Function tau (e.g., Fast Bilateral solver).
    """
    return depth_completer(D, M, I_G)

def nerf_in_optimization(F_theta, o_u, M_u, trajectory, K, batch_size, num_steps):
    """
    Main optimization loop for NeRF-In.
    """
    optimizer = optim.Adam(F_theta.parameters(), lr=0.0001) # Typical Adam lr, not specified in S1
    
    # 1. Sample K camera positions
    O = sample_camera_positions(trajectory, K)
    
    # 2. Render initial views
    I_s_list = []
    D_s_list = []
    for o_s in O:
        rays = generate_rays(o_s)
        I_s = render_image(F_theta, rays)
        D_s = compute_depth(F_theta, rays)
        I_s_list.append(I_s)
        D_s_list.append(D_s)
        
    # 3. Transfer mask using video object segmentation (STCN)
    # M_s_list = transfer_mask(M_u, I_u, I_s_list, method="STCN")
    # Note: STCN implementation is external.
    M_s_list = [M_u] * K # Placeholder for transferred masks
    
    # 4. Generate guiding materials
    I_G_list = []
    D_G_list = []
    for I_s, M_s in zip(I_s_list, M_s_list):
        I_G = inpaint_color(I_s, M_s, inpainter="MST")
        D_G = complete_depth(I_s, M_s, I_G, depth_completer="FastBilateral")
        I_G_list.append(I_G)
        D_G_list.append(D_G)
        
    # 5. Optimization Loop
    for step in range(num_steps):
        optimizer.zero_grad()
        
        # Loss 1: Color Guiding
        # L_all_color: Only on user-chosen view o_u
        rays_u = generate_rays(o_u)
        I_u_rendered = render_image(F_theta, rays_u)
        L_all_color = torch.mean((I_u_rendered - I_G_list[0]) ** 2) # Assuming I_G_list[0] corresponds to o_u if included in O, or handled separately
        
        # L_out_color: On sampled views O, outside mask
        L_out_color = 0.0
        for i, o_s in enumerate(O):
            rays_s = generate_rays(o_s)
            I_s_rendered = render_image(F_theta, rays_s)
            M_s = M_s_list[i]
            # Masked MSE
            diff = (I_s_rendered - I_G_list[i]) * M_s
            L_out_color += torch.mean(diff ** 2)
            
        L_color = L_all_color + L_out_color
        
        # Loss 2: Depth Guiding
        L_depth = 0.0
        for i, o_s in enumerate(O):
            rays_s = generate_rays(o_s)
            D_f = compute_depth(F_theta, rays_s, mode="fine")
            D_c = compute_depth(F_theta, rays_s, mode="coarse")
            D_G = D_G_list[i]
            L_depth += torch.mean((D_f - D_G) ** 2) + torch.mean((D_c - D_G) ** 2)
            
        total_loss = L_color + L_depth
        total_loss.backward()
        optimizer.step()
        
    return F_theta
```

### Parameters
*   **K**: Number of sampled camera positions/views. Not given in the sources.
*   **Batch size**: 4,096 [S1, unofficial].
*   **Sample points (fine)**: 128 [S1, unofficial].
*   **Sample points (coarse)**: 64 [S1, unofficial].
*   **Optimizer**: Adam [S1, unofficial].
*   **Training steps for inpainting**: 50,000 steps [S1, unofficial].
*   **Hardware**: Intel i7-7800X CPU and GTX-1080 GPU [S1, unofficial].

### Edge Cases and Pitfalls
*   **Optimizing only with masked MSE**: Using only masked Mean Squared Error (MSE) on transferred masks without explicit guidance causes the unwanted object to remain in the result [S1, unofficial].
*   **Using only depth-guiding loss**: Introduces color noise outside the masked region and loses color information in flat regions [S1, unofficial].
*   **Using multiple inpainted images as color guidance**: Introduces visual inconsistency between views [S1, unofficial].
*   **Guidance Artifacts**: If the initial guidance materials (from $\rho$ and $\tau$) contain artifacts, the optimized NeRF may output undesired results [S1, unofficial].

## Operating characteristics

*   **Time Complexity**: Not given in the sources. The source notes that inpainting each scene takes about five hours on the specified hardware [S1, unofficial].
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Tested on the LLFF dataset and a custom dataset with 24 camera positions per scene [S1, unofficial].
*   **Strengths**:
    *   Does not require category-specific training data or procedures [S1, unofficial].
    *   Generates visually plausible inpainted results with correct geometry and color consistency across views [S1, unofficial].
    *   Removes unwanted objects in regions with high depth variations [S1, unofficial].
*   **Limitations**:
    *   Relies on the quality of external 2D inpainting and depth completion methods [S1, unofficial].
    *   May struggle with large holes where color and texture variance is high (inherited from image inpainting limitations) [S1, unofficial].
    *   Baseline methods (per-view color updating or masked MSE retraining) result in incorrect depth maps, visual inconsistencies, or loss of fine structures [S1, unofficial].

## Background

*   **Origin**: The method was introduced by Hao-Kang Liu, I-Chao Shen, and Bing-Yu Chen in the paper "NeRF-In: Free-Form NeRF Inpainting with RGB-D Priors" [S1, unofficial].
*   **Date**: June 10, 2022 (arXiv:2206.04901v1) [S1, unofficial].
*   **History**: It is described as the first framework enabling users to remove unwanted objects from a pre-trained NeRF without category-specific data [S1, unofficial]. Previous methods required learning additional category-level conditional radiance fields or segmentation networks [S1, unofficial].
*   **Usage**: Implemented in PyTorch and Python 3.9 [S1, unofficial].

## Sources

[S1] NeRF-In: Free-Form NeRF Inpainting with RGB-D Priors, https://doi.org/10.48550/arxiv.2206.04901, fetched 2026-10-26 (unofficial)
