# NeRF-In Inpainting Optimization

Also called: Free-Form NeRF Inpainting with RGB-D Priors, NeRF object removal, NeRF-In

## Overview
NeRF-In is a method for removing unwanted objects from a pre-trained Neural Radiance Field (NeRF) by optimizing the network parameters $\Theta$ to fit new constraints derived from 2D RGB-D priors [S1, unofficial]. Unlike methods requiring category-specific training data or on-the-fly masking, NeRF-In updates the global model parameters to jointly inpaint color and geometry across multiple views, ensuring view consistency without retraining from scratch [S1, unofficial]. It sets itself apart by combining a color-guiding loss with a depth-guiding loss to prevent the "ghosting" of unwanted objects and color noise in flat regions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model $F_\Theta$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$; a user-chosen camera position $\mathbf{o}_u$; a user-drawn binary mask $M_u$ on the rendered view $I_u = F_{\Theta_{image}}(\mathbf{o}_u)$, where $M_u=1$ indicates pixels outside the masked region (background) and $M_u=0$ inside (object to remove) [S1, unofficial].
*   **Output:** Updated NeRF parameters $\tilde{\Theta}$ [S1, unofficial].

**Data Structures:**
*   **NeRF Model $F_\Theta$:** A multi-layer perceptron (MLP) taking 3D coordinates and viewing directions. The specific architecture (layer sizes, activation functions) is not detailed in the source, but it follows the standard NeRF formulation [S1, unofficial].
*   **Ray Batches:** Sets of rays $\mathbf{r}$ sampled from camera positions. Each ray is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Sample Points:** For each ray, $N$ quadrature points are sampled between near plane $t_n$ and far plane $t_f$. The source specifies $N_{coarse}=64$ and $N_{fine}=128$ [S1, unofficial].
*   **Guiding Materials:**
    *   $I^G$: Guiding color images.
    *   $D^G$: Guiding depth images.
    *   $M$: Transferred masks for sampled views.

**Numbered Steps:**

1.  **User Input:** The user selects a view $\mathbf{o}_u$ from the pre-trained NeRF and draws a free-form mask $M_u$ specifying the region to remove [S1, unofficial].
2.  **View Sampling:** Sample $K$ camera positions $\mathcal{O} = \{\mathbf{o}_s \mid s=1\dots K\}$ along a pre-set trajectory (e.g., LLFF test-set trajectory) [S1, unofficial].
3.  **Initial Rendering:** Render RGB images $I_s$ and depth images $D_s$ for all sampled views $\mathbf{o}_s$ using the current NeRF $F_\Theta$ [S1, unofficial].
4.  **Mask Transfer:** Transfer the user-drawn mask $M_u$ to the sampled views to generate transferred masks $M = \{M_s \mid s=1\dots K\}$ using a video object segmentation method (e.g., STCN) [S1, unofficial].
5.  **Guiding Material Generation:**
    *   Generate guiding color images $I^G = \{I_s^G \mid s=1\dots K\}$ using an image inpainting method $\rho$: $I_s^G = \rho(I_s, M_s)$ [S1, unofficial].
    *   Generate guiding depth images $D^G = \{D_s^G \mid s=1\dots K\}$ using a depth completion method $\tau$: $D_s^G = \tau(D_s, M_s, I_s^G)$ [S1, unofficial].
    *   *Note:* The source specifies using MST inpainting network for $\rho$ and Fast Bilateral Solver for $\tau$ [S1, unofficial].
6.  **Loss Definition:** Define the optimization objective as the sum of color-guiding loss $L_{color}(\Theta)$ and depth-guiding loss $L_{depth}(\Theta)$ [S1, unofficial].
7.  **Optimization:** Optimize $\Theta$ to obtain $\tilde{\Theta}$ by minimizing the total loss using the Adam optimizer for 50,000 steps [S1, unofficial].

**Formulas:**

**Volume Rendering:**
The expected color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ is computed via numerical quadrature [S1, unofficial]:
$$ \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T(t_i) (1 - \exp(-\sigma(t_i)\delta_i)) c(t_i) $$
where $T(t_i) = \exp(-\sum_{j=1}^{i-1} \sigma(t_j)\delta_j)$, $N$ is the number of quadrature points, $\delta_i = t_{i+1} - t_i$, and $c(t_i), \sigma(t_i)$ are the color and density predicted by $F_\Theta$ at point $t_i$ [S1, unofficial].

**Depth Rendering:**
Rendered depth $D(\mathbf{o}_s)$ is computed by accumulating density $\sigma$ from ray batches during volume rendering [S1, unofficial]. Specifically, $D_f(\mathbf{o}_s)$ and $D_c(\mathbf{o}_s)$ denote the fine and coarse volume predicted depth images, respectively [S1, unofficial].

**Color-Guiding Loss:**
The total color loss is the sum of an all-view loss and an outside-view loss [S1, unofficial]:
$$ L_{color}(\Theta) = L_{all\_color}(\Theta) + L_{out\_color}(\Theta) $$
where [S1, unofficial]:
$$ L_{all\_color}(\Theta) = \sum_{\mathbf{o} \in \mathcal{O}_{all}} \| F_{\Theta_{image}}(\mathbf{o}) - I_{\mathbf{o}}^G \|_2^2 $$
$$ L_{out\_color}(\Theta) = \sum_{\mathbf{o} \in \mathcal{O}_{out}} \| (F_{\Theta_{image}}(\mathbf{o}) - I_{\mathbf{o}}^G) \odot M_{\mathbf{o}} \|_2^2 $$
The source sets $\mathcal{O}_{all} = \{\mathbf{o}_u\}$ (only the user-chosen view) and $\mathcal{O}_{out} = \mathcal{O}$ (all sampled views) [S1, unofficial]. $M_{\mathbf{o}}$ is the transferred mask for view $\mathbf{o}$ [S1, unofficial].

**Depth-Guiding Loss:**
The depth loss measures the L2 distance between rendered depths and guiding depths for all sampled views [S1, unofficial]:
$$ L_{depth}(\Theta) = \sum_{\mathbf{o}_s \in \mathcal{O}} \left( \| D_f(\mathbf{o}_s) - D_{\mathbf{o}_s}^G \|_2^2 + \| D_c(\mathbf{o}_s) - D_{\mathbf{o}_s}^G \|_2^2 \right) $$
where $D_f$ and $D_c$ are the fine and coarse depth predictions [S1, unofficial].

**Total Objective:**
$$ \tilde{\Theta} := \arg \min_{\Theta} \left( L_{color}(\Theta) + L_{depth}(\Theta) \right) $$
[S1, unofficial]

**Parameters:**
*   $K$: Number of sampled camera positions. Not explicitly given, but implied to be a small set along a trajectory [S1, unofficial].
*   Batch size: 4,096 [S1, unofficial].
*   Sample points (fine): 128 [S1, unofficial].
*   Sample points (coarse): 64 [S1, unofficial].
*   Optimizer: Adam [S1, unofficial].
*   Training steps for inpainting: 50,000 [S1, unofficial].
*   Hardware: Intel i7-7800X CPU, GTX-1080 GPU [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def volume_render_color(ray_origins, ray_directions, nerf_model, num_samples):
    """
    Renders color for a batch of rays using the NeRF model.
    ray_origins: (B, 3)
    ray_directions: (B, 3)
    nerf_model: The NeRF MLP F_Theta
    num_samples: N, number of quadrature points
    Returns:
        colors: (B, 3)
        depths: (B,)
        weights: (B, N) for potential depth calculation if needed
    """
    # Sample points along rays
    t_vals = torch.linspace(0., 1., num_samples, device=ray_origins.device)
    # Assuming near/far planes are handled by the model or fixed bounds
    # In standard NeRF, t_vals are mapped to t_n and t_f. 
    # Here we assume a simplified linear sampling or that the model handles bounds.
    # For strict implementation, t_vals should be interpolated between t_n and t_f.
    # Let's assume t_vals are already in [0, 1] relative to ray length or bounds are fixed.
    # Points: o + t * d
    points = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_vals.unsqueeze(0).unsqueeze(2) # (B, N, 3)
    
    # Flatten for MLP
    points_flat = points.reshape(-1, 3)
    
    # Get directions for each point (same for all points on a ray)
    dirs_flat = ray_directions.unsqueeze(1).expand(-1, num_samples, -1).reshape(-1, 3)
    
    # Query NeRF
    # F_Theta(x, d) -> (sigma, c)
    # Note: Standard NeRF uses positional encoding. This is assumed to be inside nerf_model.
    sigma_flat, color_flat = nerf_model(points_flat, dirs_flat)
    
    sigma = sigma_flat.reshape(-1, num_samples)
    color = color_flat.reshape(-1, num_samples, 3)
    
    # Compute deltas
    # Assuming uniform sampling for simplicity or pre-computed t_vals
    deltas = t_vals[1:] - t_vals[:-1] # (N-1,)
    # Add a small epsilon to avoid division by zero or log(0) issues if needed, 
    # but here we use exp(-sigma * delta)
    
    # Compute transmittance
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # alpha_i = 1 - exp(-sigma_i * delta_i)
    # w_i = T_i * alpha_i
    
    # Using cumulative sum for efficiency
    # sigma * delta
    sigma_delta = sigma * deltas.unsqueeze(0) # (B, N-1)
    # Note: The formula in source uses N points. Usually N-1 deltas.
    # Let's align with standard NeRF implementation:
    # t_vals has N points. deltas has N-1.
    # We need to handle the last segment or assume N points define N-1 intervals.
    # Source says N quadrature points.
    
    # Standard implementation:
    # alpha = 1 - torch.exp(-sigma * deltas)
    # T = torch.cumsum(alpha, dim=1) ... no, T is product of (1-alpha)
    # T_i = prod_{j=1}^{i-1} (1 - alpha_j)
    
    # Let's compute weights directly
    # alpha = 1 - exp(-sigma * deltas)
    # weights = alpha * prod(1-alpha)
    
    # To avoid numerical instability, use log space or careful accumulation
    # But for pseudocode clarity:
    alpha = 1.0 - torch.exp(-sigma * deltas.unsqueeze(0)) # (B, N-1)
    transmittance = torch.cumprod(1.0 - alpha, dim=1) # (B, N-1)
    # Shift transmittance to align with alpha for weights
    # T_1 = 1, T_2 = 1-alpha_1, etc.
    # weights_i = T_i * alpha_i
    # We need T shifted right by 1, with T_1 = 1
    ones = torch.ones_like(transmittance[:, :1])
    transmittance_shifted = torch.cat([ones, transmittance[:, :-1]], dim=1)
    weights = transmittance_shifted * alpha
    
    # Color
    colors = torch.sum(weights.unsqueeze(2) * color[:, :-1, :], dim=1) # (B, 3)
    
    # Depth
    # D = sum(w_i * t_i)
    # t_vals used for points are t_1 ... t_N. 
    # The weights correspond to intervals [t_1, t_2], ..., [t_{N-1}, t_N]?
    # Or points are centers? Source says "quadrature points".
    # Standard NeRF uses midpoints or endpoints.
    # Let's assume t_vals are the sample points t_i.
    # If weights are for intervals, we need interval centers or endpoints.
    # Source formula: sum T(ti)(1-exp(-sigma(ti)delta_i)) c(ti).
    # This implies t_i are the sample points.
    # delta_i = t_{i+1} - t_i.
    # So weights are associated with t_i for i=1..N-1?
    # Let's assume t_vals has N points, deltas has N-1.
    # The sum is over i=1 to N-1? Or N?
    # Source: "N is the number of quadrature points".
    # Formula: sum_{i=1}^N ...
    # If N points, there are N-1 intervals.
    # Usually, the last point is the far plane.
    # Let's assume the standard NeRF implementation where N samples are taken.
    
    # Re-evaluating based on source formula:
    # delta_i = t_{i+1} - t_i.
    # If i goes to N, t_{N+1} is needed?
    # Usually, N samples means N intervals if t_0 is near and t_N is far?
    # Or N points between near and far.
    # Let's stick to the standard PyTorch NeRF implementation logic:
    # t_vals = linspace(t_n, t_f, N)
    # deltas = t_vals[1:] - t_vals[:-1]
    # alpha = 1 - exp(-sigma * deltas)
    # weights = alpha * cumprod(1-alpha) shifted
    # color = sum(weights * colors)
    # depth = sum(weights * t_vals[:-1]) ? Or midpoints?
    # Source says c(t_i). So t_i are the points.
    # If weights are for t_i, then delta_i must be defined.
    # Let's assume t_vals are the sample points t_i.
    # And deltas are t_{i+1} - t_i.
    # Then the sum is i=1 to N-1.
    
    # For the purpose of this pseudocode, we will use the standard NeRF rendering function
    # which returns color and depth.
    
    return colors, depth # Depth calculation omitted for brevity, assumed standard

def nerf_in_inpainting(nerf_model, user_view_o_u, user_mask_M_u, trajectory_views, 
                       inpainting_method_rho, depth_completion_method_tau, 
                       num_steps=50000, batch_size=4096):
    """
    Optimizes NeRF parameters to remove object specified by user_mask_M_u.
    """
    optimizer = optim.Adam(nerf_model.parameters(), lr=0.0001) # Typical LR not given, Adam is used
    
    # 1. Render initial views
    # Sample K views from trajectory
    sampled_views = trajectory_views[:len(trajectory_views)] # K views
    
    # Render initial RGB and Depth for sampled views
    # Note: Rendering depth requires accumulating sigma.
    initial_images = {}
    initial_depths = {}
    for o in sampled_views:
        rays = generate_rays(o) # Helper to generate rays for camera o
        # Render coarse and fine if using two-stage, but here we just need initial guidance
        # We need to render to get I_s and D_s
        I_s, D_s = render_view(nerf_model, o) 
        initial_images[o] = I_s
        initial_depths[o] = D_s
        
    # 2. Transfer Mask
    # Use STCN or similar video object segmentation
    transferred_masks = {}
    transferred_masks[user_view_o_u] = user_mask_M_u
    for o in sampled_views:
        if o != user_view_o_u:
            # Transfer mask from user_view_o_u to o
            M_s = transfer_mask(user_mask_M_u, user_view_o_u, o, method="STCN")
            transferred_masks[o] = M_s
            
    # 3. Generate Guiding Materials
    guiding_images = {}
    guiding_depths = {}
    
    for o in sampled_views:
        I_s = initial_images[o]
        D_s = initial_depths[o]
        M_s = transferred_masks[o]
        
        # Generate guiding color
        I_s_G = inpainting_method_rho(I_s, M_s)
        guiding_images[o] = I_s_G
        
        # Generate guiding depth
        D_s_G = depth_completion_method_tau(D_s, M_s, I_s_G)
        guiding_depths[o] = D_s_G
        
    # 4. Optimization Loop
    for step in range(num_steps):
        # Sample a batch of rays
        # The source mentions optimizing Eq 7.
        # We need to compute L_color and L_depth.
        
        # Define sets
        O_all = [user_view_o_u]
        O_out = sampled_views # All other views
        
        # Collect rays for O_all and O_out
        # We need to render these views with the current NeRF
        # And compare against guiding images/depths
        
        # Render for O_all
        rays_all = []
        targets_all = []
        masks_all = []
        
        for o in O_all:
            rays = generate_rays(o)
            # Render color
            colors = volume_render_color(rays.origins, rays.directions, nerf_model, num_samples=128)
            # Target is guiding image
            target = guiding_images[o]
            # Mask is transferred mask
            mask = transferred_masks[o]
            
            # L_all_color: Full image difference
            # L_out_color: Outside mask difference
            # We can compute these in one pass or separate.
            # Let's compute loss directly.
            pass 
            
        # Render for O_out
        rays_out = []
        for o in O_out:
            rays = generate_rays(o)
            rays_out.append(rays)
            
        # Compute Losses
        # L_all_color: sum over O_all of || F(o) - I_G ||^2
        # L_out_color: sum over O_out of || (F(o) - I_G) * M ||^2
        # L_depth: sum over O (all sampled views) of || D_f - D_G ||^2 + || D_c - D_G ||^2
        
        # Implementation detail:
        # We need to render both color and depth for all sampled views.
        
        total_loss = 0.0
        
        # L_all_color
        for o in O_all:
            rays = generate_rays(o)
            colors = volume_render_color(rays.origins, rays.directions, nerf_model, num_samples=128)
            target = guiding_images[o]
            # Reshape colors to match image dimensions
            loss = torch.mean((colors - target)**2)
            total_loss += loss
            
        # L_out_color
        for o in O_out:
            rays = generate_rays(o)
            colors = volume_render_color(rays.origins, rays.directions, nerf_model, num_samples=128)
            target = guiding_images[o]
            mask = transferred_masks[o]
            # Element-wise multiplication with mask
            diff = (colors - target) * mask
            loss = torch.mean(diff**2)
            total_loss += loss
            
        # L_depth
        for o in sampled_views:
            rays = generate_rays(o)
            # Render coarse and fine depth
            # Assuming nerf_model can return depth or we have a separate depth renderer
            D_f = render_depth_fine(rays, nerf_model, num_samples=128)
            D_c = render_depth_coarse(rays, nerf_model, num_samples=64)
            target_D = guiding_depths[o]
            
            loss_f = torch.mean((D_f - target_D)**2)
            loss_c = torch.mean((D_c - target_D)**2)
            total_loss += loss_f + loss_c
            
        # Backprop
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
        
    return nerf_model
```

**Edge Cases and Pitfalls:**
*   **Masked MSE Only:** Optimizing only with masked MSE on transferred masks without explicit guidance causes the unwanted object to remain in the result [S1, unofficial].
*   **Depth Loss Only:** Using only depth-guiding loss introduces color noise outside the masked region and loses color information in flat regions [S1, unofficial].
*   **Multiple Color Guidance:** Using multiple inpainted images as color guidance introduces visual inconsistency between views. The method restricts color guidance to the user-chosen view only [S1, unofficial].
*   **Guidance Artifacts:** If the initial guidance materials (from $\rho$ and $\tau$) have artifacts, the optimized NeRF may output undesired results [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The optimization takes approximately 5 hours per scene on an Intel i7-7800X CPU and GTX-1080 GPU for 50,000 steps [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Scaling:** The method scales with the number of sampled views $K$ and the batch size (4,096 rays) [S1, unofficial].
*   **Strengths:** No category-specific training data required; preserves view consistency; removes objects without leaving ghosts [S1, unofficial].
*   **Limitations:** Relies on the quality of external inpainting/depth completion methods; may inherit artifacts from guidance [S1, unofficial].
*   **When to use:** When removing unwanted objects from a pre-trained NeRF without access to category-specific data or the ability to retrain from scratch [S1, unofficial].
*   **When not to use:** When high-fidelity texture synthesis is required beyond the capability of the 2D inpainting prior, or when real-time editing is needed [S1, unofficial].

## Background
NeRF-In was introduced by Hao-Kang Liu, I-Chao Shen, and Bing-Yu Chen in 2022 [S1, unofficial]. It is the first framework to enable free-form inpainting on NeRFs without category-specific training data [S1, unofficial]. It builds on the principles of differentiable volume rendering and 2D image inpainting techniques [S1, unofficial].

## Sources
[S1] NeRF-In: Free-Form NeRF Inpainting with RGB-D Priors, https://doi.org/10.48550/arxiv.2206.04901, fetched 2026-10-26 (unofficial)
