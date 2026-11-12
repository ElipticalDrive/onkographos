# NeRF-In Inpainting Optimization

Also called: Free-Form NeRF Inpainting with RGB-D Priors, NeRF object removal, NeRF-In

## Overview
NeRF-In is a method for removing unwanted objects from a pre-trained Neural Radiance Field (NeRF) by optimizing the model parameters $\Theta$ using color and depth guidance derived from 2D images [S1, unofficial]. It differs from sibling methods by explicitly combining a color-guiding loss with a depth-guiding loss to ensure both visual plausibility and geometric consistency across views, avoiding the artifacts common in methods that rely solely on color or masked retraining [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A pre-trained NeRF model $F_\Theta$ mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $c$; a user-chosen camera position $o_u$; a user-drawn mask $M_u$ on the rendered view $I_u = F_{\Theta_{image}}(o_u)$, where $M_u=1$ for pixels outside the masked region (background/valid) and $M_u=0$ inside (object to remove) [S1, unofficial].
*   **Output:** Updated NeRF parameters $\tilde{\Theta}$ [S1, unofficial].

**Data Structures**
*   **NeRF Model $F_\Theta$:** A multi-layer perceptron (MLP) taking position and direction as input, outputting density $\sigma$ and color $c$. The specific architecture (layer sizes, activation functions) is not detailed in the source, but it follows the standard NeRF formulation [S1, unofficial].
*   **Ray Representation:** A camera ray is defined as $r(t) = o + t\mathbf{d}$, where $o$ is the camera origin and $\mathbf{d}$ is the unit direction vector [S1, unofficial].
*   **Sampling Points:** For each ray, $N$ quadrature points are sampled between near plane $t_n$ and far plane $t_f$. The distance between adjacent points is $\delta_i = t_{i+1} - t_i$ [S1, unofficial].
    *   Coarse model uses 64 sample points [S1, unofficial].
    *   Fine model uses 128 sample points [S1, unofficial].

**Numbered Steps**
1.  **Initialization:** Initialize the NeRF model parameters $\Theta$ (using a pre-trained model) [S1, unofficial].
2.  **View Sampling:** Sample $K$ camera positions $O = \{o_s \mid s = 1 \dots K\}$ along a pre-set trajectory (e.g., LLFF test-set trajectory) [S1, unofficial].
3.  **Rendering:** Render RGB images $I_s$ and depth images $D_s$ for each sampled camera position $o_s \in O$ using the current NeRF model $F_\Theta$ [S1, unofficial].
4.  **Mask Transfer:** Transfer the user-drawn mask $M_u$ from the user-chosen view $I_u$ to the sampled views $I_s$ to generate transferred masks $M = \{M_s \mid s = 1 \dots K\}$ using a video object segmentation method (e.g., STCN) [S1, unofficial].
5.  **Guiding Material Generation:**
    *   Generate guiding color images $I^G = \{I_s^G \mid s = 1 \dots K\}$ using an image inpainting method $\rho$: $I_s^G = \rho(I_s, M_s)$ [S1, unofficial].
    *   Generate guiding depth images $D^G = \{D_s^G \mid s = 1 \dots K\}$ using a depth completion method $\tau$: $D_s^G = \tau(D_s, M_s, I_s^G)$ [S1, unofficial].
6.  **Optimization:** Optimize the NeRF parameters $\Theta$ to obtain $\tilde{\Theta}$ by minimizing the joint inpainting formulation $\Phi(M_u, I_u^G, M, D_s^G)$, which consists of the color-guiding loss $L_{color}(\Theta)$ and the depth-guiding loss $L_{depth}(\Theta)$ [S1, unofficial].

**Formulas**
*   **Volume Rendering (Color):** The expected color $\hat{C}(r)$ of a camera ray $r$ is computed via numerical quadrature [S1, unofficial]:
    $$ \hat{C}(r) = \sum_{i=1}^{N} T(t_i) (1 - \exp(-\sigma(t_i)\delta_i)) c(t_i) $$
    where $T(t_i) = \exp(-\sum_{j=1}^{i-1} \sigma(t_j)\delta_j)$ is the accumulated transmittance, $c(t_i)$ and $\sigma(t_i)$ are the color and density at point $t_i$, and $N$ is the number of quadrature points [S1, unofficial].

*   **Depth Rendering:** The rendered depth $D(o_s)$ is computed by accumulating density $\sigma$ from ray batches during volume rendering. Specifically, $D_f(o_s)$ is the fine volume predicted depth and $D_c(o_s)$ is the coarse volume predicted depth [S1, unofficial].

*   **Color-Guiding Loss:** The total color-guiding loss is the sum of the all-view color loss and the outside-view color loss [S1, unofficial]:
    $$ L_{color}(\Theta) = L_{all\_color}(\Theta) + L_{out\_color}(\Theta) $$
    where [S1, unofficial]:
    $$ L_{all\_color}(\Theta) = \sum_{o \in O_{all}} \| F_{\Theta_{image}}(o) - I_o^G \|_2^2 $$
    $$ L_{out\_color}(\Theta) = \sum_{o \in O_{out}} \| (F_{\Theta_{image}}(o) - I_o^G) \odot M_o \|_2^2 $$
    In the main framework, $O_{all} = \{o_u\}$ (the user-chosen view) and $O_{out} = O$ (all other sampled views) [S1, unofficial]. Note: The source defines $L_{out\_color}$ using the mask $M_o$ which is 1 outside the masked region, effectively weighting the loss only on the background/valid pixels [S1, unofficial].

*   **Depth-Guiding Loss:** The depth-guiding loss measures the L2 distance between rendered fine/coarse depth images and guiding depth [S1, unofficial]:
    $$ L_{depth}(\Theta) = \sum_{o_s \in O} \left( \| D_f(o_s) - D_s^G \|_2^2 + \| D_c(o_s) - D_s^G \|_2^2 \right) $$

*   **Total Loss:**
    $$ \tilde{\Theta} := \arg \min_{\Theta} L_{color}(\Theta) + L_{depth}(\Theta) $$

**Parameters**
*   $K$: Number of sampled camera positions/views. Not given in the sources.
*   Batch size: 4,096 [S1, unofficial].
*   Optimizer: Adam [S1, unofficial].
*   Training steps for inpainting: 50,000 steps [S1, unofficial].
*   Sample points (coarse): 64 [S1, unofficial].
*   Sample points (fine): 128 [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def volume_render_color(ray_origins, ray_directions, nerf_model, num_samples):
    """
    Renders color for a batch of rays using the NeRF model.
    Returns: predicted_color (B, 3)
    """
    # Sample points along rays
    t_vals = torch.linspace(0., 1., num_samples).to(ray_origins.device)
    t_vals = t_vals.unsqueeze(0).expand(ray_origins.shape[0], num_samples)
    # Add noise for anti-aliasing (standard NeRF practice, implied by quadrature)
    # Note: Exact noise addition not specified in source, but standard in NeRF [G]
    
    # Compute ray points
    ray_points = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_vals.unsqueeze(2)
    
    # Query NeRF for density and color
    # nerf_model takes (position, direction)
    # direction is constant for all points on a ray
    directions = ray_directions.unsqueeze(1).expand(ray_points.shape)
    
    sigma = nerf_model.get_density(ray_points, directions) # Shape: (B, N)
    color = nerf_model.get_color(ray_points, directions)   # Shape: (B, N, 3)
    
    # Compute transmittance
    deltas = t_vals[:, 1:] - t_vals[:, :-1]
    # Handle first delta (often 0 or near plane distance)
    # Standard NeRF implementation details [G]
    
    alpha = 1.0 - torch.exp(-sigma * deltas)
    T = torch.cumprod(torch.exp(-sigma * deltas), dim=1)
    T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=1)
    
    weights = alpha * T
    predicted_color = torch.sum(weights.unsqueeze(2) * color, dim=1)
    
    return predicted_color

def render_depth(ray_origins, ray_directions, nerf_model, num_samples):
    """
    Renders depth for a batch of rays.
    Returns: predicted_depth (B,)
    """
    # Reuse t_vals and ray_points from volume_render_color logic
    # Compute density
    directions = ray_directions.unsqueeze(1).expand(ray_points.shape)
    sigma = nerf_model.get_density(ray_points, directions)
    
    # Compute transmittance (same as above)
    deltas = t_vals[:, 1:] - t_vals[:, :-1]
    alpha = 1.0 - torch.exp(-sigma * deltas)
    T = torch.cumprod(torch.exp(-sigma * deltas), dim=1)
    T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=1)
    
    # Depth is weighted average of t_vals
    weights = alpha * T
    predicted_depth = torch.sum(weights * t_vals, dim=1)
    
    return predicted_depth

def nerf_in_optimization(nerf_model, user_view_origin, user_view_direction, user_mask, 
                         sampled_views, inpainting_method, depth_completion_method, 
                         num_coarse_samples, num_fine_samples, num_steps):
    """
    Optimizes NeRF parameters to remove unwanted objects.
    """
    optimizer = optim.Adam(nerf_model.parameters(), lr=0.0001) # Typical LR [G]
    
    # 1. Render initial views
    # Note: sampled_views contains (origin, direction) pairs
    initial_colors = {}
    initial_depths = {}
    for view in sampled_views:
        o, d = view
        # Render coarse and fine for depth calculation if needed, 
        # but source implies simple rendering for guidance generation
        # We need RGB and Depth for guidance generation
        c = volume_render_color(o, d, nerf_model, num_coarse_samples)
        d_val = render_depth(o, d, nerf_model, num_coarse_samples)
        initial_colors[o] = c
        initial_depths[o] = d_val
        
    # 2. Transfer Mask
    # Use STCN or similar video object segmentation
    transferred_masks = transfer_mask(user_mask, user_view_origin, sampled_views)
    
    # 3. Generate Guidance
    guiding_colors = {}
    guiding_depths = {}
    for view in sampled_views:
        o = view.origin
        # Inpaint color
        I_s = initial_colors[o] # This is a tensor, inpainting method expects image
        # Convert tensor to image format for inpainting method [G]
        I_s_img = tensor_to_image(I_s)
        M_s = transferred_masks[o]
        
        I_s_G = inpainting_method(I_s_img, M_s)
        guiding_colors[o] = image_to_tensor(I_s_G)
        
        # Inpaint depth
        D_s = initial_depths[o]
        D_s_img = tensor_to_image(D_s)
        D_s_G = depth_completion_method(D_s_img, M_s, I_s_G)
        guiding_depths[o] = image_to_tensor(D_s_G)
        
    # 4. Optimization Loop
    for step in range(num_steps):
        optimizer.zero_grad()
        
        # Define sets
        O_all = [user_view_origin]
        O_out = [v.origin for v in sampled_views]
        
        # Compute L_all_color (on user view)
        # Render user view
        user_color = volume_render_color(user_view_origin, user_view_direction, nerf_model, num_fine_samples)
        I_u_G = guiding_colors[user_view_origin]
        loss_all_color = torch.mean((user_color - I_u_G) ** 2)
        
        # Compute L_out_color (on other views, masked)
        loss_out_color = 0.0
        for o in O_out:
            # Render view
            view = next(v for v in sampled_views if v.origin == o)
            out_color = volume_render_color(view.origin, view.direction, nerf_model, num_fine_samples)
            I_o_G = guiding_colors[o]
            M_o = transferred_masks[o]
            
            # Masked MSE: only compute loss on pixels where M_o = 1 (outside mask)
            # Element-wise multiplication with mask
            masked_diff = (out_color - I_o_G) * M_o
            loss_out_color += torch.mean(masked_diff ** 2)
            
        L_color = loss_all_color + loss_out_color
        
        # Compute L_depth (on other views)
        loss_depth = 0.0
        for o in O_out:
            view = next(v for v in sampled_views if v.origin == o)
            # Render fine and coarse depth
            D_f = render_depth(view.origin, view.direction, nerf_model, num_fine_samples)
            D_c = render_depth(view.origin, view.direction, nerf_model, num_coarse_samples)
            D_G = guiding_depths[o]
            
            loss_depth += torch.mean((D_f - D_G) ** 2)
            loss_depth += torch.mean((D_c - D_G) ** 2)
            
        L_depth = loss_depth
        
        # Total Loss
        total_loss = L_color + L_depth
        total_loss.backward()
        optimizer.step()
        
    return nerf_model
```

**Edge Cases and Pitfalls**
*   **Optimizing only with masked MSE:** If the optimization uses only masked MSE on transferred masks without explicit guidance, the unwanted object remains in the result [S1, unofficial].
*   **Using only depth-guiding loss:** This introduces color noise outside the masked region and loses color information in flat regions [S1, unofficial].
*   **Using multiple inpainted images as color guidance:** This introduces visual inconsistency between views [S1, unofficial]. The method restricts color guidance to the user-chosen view only to maintain consistency.
*   **Guidance Artifacts:** If the initial guidance materials (from inpainting/depth completion) contain artifacts, the optimized NeRF may output undesired results [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The optimization takes approximately 5 hours per scene on an Intel i7-7800X CPU and GTX-1080 GPU for 50,000 steps [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Does not require category-specific training data or procedures; removes unwanted objects while preserving view consistency and geometry [S1, unofficial].
*   **Limitations:** Relies on the quality of external 2D inpainting and depth completion methods; may propagate artifacts from these methods into the 3D model [S1, unofficial].
*   **When to use:** When removing specific objects from a pre-trained NeRF without access to the original training data or category-specific models.
*   **When not to use:** When high-quality depth guidance is unavailable or when the scene contains large flat regions where depth-only guidance fails to preserve color [S1, unofficial].

## Background
NeRF-In was introduced by Hao-Kang Liu, I-Chao Shen, and Bing-Yu Chen in 2022 [S1, unofficial]. It is the first framework to enable free-form inpainting on NeRF without category-specific data or training. It builds on the standard NeRF volume rendering formulation and integrates 2D priors (RGB-D) to guide the 3D optimization [S1, unofficial].

## Sources
[S1] NeRF-In: Free-Form NeRF Inpainting with RGB-D Priors, https://doi.org/10.48550/arxiv.2206.04901, fetched 2026-10-26 (unofficial)
