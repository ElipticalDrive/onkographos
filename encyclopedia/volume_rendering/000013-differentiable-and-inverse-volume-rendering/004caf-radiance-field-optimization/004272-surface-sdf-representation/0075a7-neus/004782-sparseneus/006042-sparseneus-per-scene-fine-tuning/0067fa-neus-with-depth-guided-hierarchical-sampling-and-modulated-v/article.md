# NeuS with Depth-Guided Hierarchical Sampling and Modulated Variance

Also called: NeuS with GS depth guide, instant_nsr NeuS

## Overview
This method is a neural rendering technique that renders volumetric alpha from a signed distance field (SDF) using a learnable inverse temperature (variance) parameter, with hierarchical ray sampling guided by predicted depth from a 3D Gaussian Splatting (3DGS) branch [S1, unofficial]. It differs from standard NeuS by replacing fixed uniform sampling with depth-guided hierarchical sampling, using GS-predicted depth to define adaptive sampling intervals proportional to local SDF magnitude, thereby concentrating samples near surfaces and reducing empty-space queries [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera pose (origin $\mathbf{o}$, direction $\mathbf{d}$), scene bounding box (AABB), 3DGS depth prediction $z_{\text{GS}}$, SDF network $S(\mathbf{p})$, VarianceNetwork.
*   **Outputs:** Rendered color $C$, accumulated alpha $\alpha_{\text{acc}}$, depth $D$.

**Data Structures:**
*   `scene_aabb`: `torch.float32` tensor of shape `[6]`, values `[-radius, -radius, -radius, radius, radius, radius]`.
*   `occupancy_grid`: `nerfacc.OccupancyGrid` instance, `roi_aabb=scene_aabb`, `resolution=256`, `contraction_type=AABB`.
*   `occupancy_grid_bg`: `nerfacc.OccupancyGrid` instance, `roi_aabb=scene_aabb`, `resolution=128`, `contraction_type=UN_BOUNDED_SPHERE`.
*   `dists`: `torch` tensor of shape `[N_rays, N_samples]`, containing sorted sample distances along rays.
*   `positions`: `torch` tensor of shape `[N_rays * N_samples, 3]`, computed as $\mathbf{o} + t_i \mathbf{d}$.

**Parameters:**
*   `init_val`: Initial value for the learnable variance parameter $v$ in VarianceNetwork.
*   `modulate`: Boolean flag enabling linear modulation of the inverse temperature ($\text{inv\_s}$) during training.
*   `mod_start_steps`: Global step at which variance modulation begins.
*   `reach_max_steps`: Global step at which variance modulation reaches its maximum bound.
*   `max_inv_s`: Upper bound for the modulated inverse temperature value.
*   `radius`: Scene bounding box half-extent.
*   `num_samples_per_ray`: Total number of sample points per ray for coarse/fine sampling.
*   `grid_prune_occ_thre`: Occupancy grid pruning threshold (default 0.01).
*   `cos_anneal_end`: Global step at which cosine annealing for surface normal alignment ends (default 0 means disabled).
*   `num_samples_equispaced`: Number of equispaced samples in the coarse sampling pass.
*   `num_samples_full`: Number of equispaced samples when depth guidance is disabled.
*   `epsilon_fd`: Finite difference epsilon (standard value $10^{-5}$ or similar, used for clamping).

**Steps:**

1.  **Initialization:**
    *   Initialize `VarianceNetwork` with learnable parameter $v = \text{init\_val}$.
    *   Compute initial inverse temperature $\sigma = \exp(10 \cdot v)$.
    *   Set `mod_val` = $\sigma$.
    *   Setup geometry/texture modules. Set contraction type to AABB.
    *   Compute `render_step_size` as $1.732 \times 2 \times \text{radius} / \text{num\_samples\_per\_ray}$.
    *   Initialize `occupancy_grid` at resolution 256 and `occupancy_grid_bg` at resolution 128.

2.  **Occupancy Grid Update (Every N steps):**
    *   Query SDF at grid points.
    *   Compute previous/next SDF offsets by $\pm \Delta t/2$, where $\Delta t = \text{render\_step\_size}$.
    *   Apply sigmoid with $\text{inv\_s} = 1/\sigma$.
    *   Derive alpha via CDF difference.
    *   Update occupancy grid if alpha > `grid_prune_occ_thre`.

3.  **Ray Sampling (Depth-Guided Hierarchical):**
    *   Compute AABB intersection $[t_{\text{min}}^{\text{AABB}}, t_{\text{max}}^{\text{AABB}}]$.
    *   **Coarse Bounds:**
        *   If GS depth $z_{\text{GS}}$ is provided:
            *   Query SDF at depth position $\mathbf{p}_{\text{depth}} = \mathbf{o} + z_{\text{GS}} \mathbf{d}$. Let $s_{\text{depth}} = S(\mathbf{p}_{\text{depth}})$.
            *   Compute interval $\text{interval} = |s_{\text{depth}}| \cdot \text{radius} \cdot 3$.
            *   Clamp interval to $[\epsilon_{\text{fd}} \cdot 16, \infty)$.
            *   Set coarse bounds $[t_{\text{min}}^{\text{coarse}}, t_{\text{max}}^{\text{coarse}}] = [z_{\text{GS}} - \text{interval}, z_{\text{GS}} + \text{interval}]$.
        *   Else (no GS depth):
            *   Set coarse bounds to full AABB range.
    *   **Fine Bounds:**
        *   If GS depth is provided:
            *   Compute fine interval $\text{interval}_{\text{fine}} = |s_{\text{depth}}| \cdot \text{radius}$.
            *   Clamp to $[\epsilon_{\text{fd}} \cdot 8, \infty)$.
            *   Set fine bounds $[t_{\text{min}}^{\text{fine}}, t_{\text{max}}^{\text{fine}}] = [z_{\text{GS}} - \text{interval}_{\text{fine}}, z_{\text{GS}} + \text{interval}_{\text{fine}}]$.
        *   Else:
            *   Set fine bounds to full AABB range.
    *   **Clamping:**
        *   $t_{\text{min}} = \max(\text{low\_bound}, t_{\text{min}}^{\text{coarse}}, t_{\text{min}}^{\text{AABB}})$.
        *   $t_{\text{max}} = \min(\text{upper\_bound}, t_{\text{max}}^{\text{coarse}}, t_{\text{max}}^{\text{AABB}})$.
        *   If $t_{\text{min}} \ge t_{\text{max}}$, discard ray.
        *   For fine pass: $t_{\text{min}}^{\text{fine}} = \max(\text{low\_bound}^{\text{fine}}, t_{\text{min}}^{\text{fine}}, t_{\text{min}}^{\text{AABB}})$, $t_{\text{max}}^{\text{fine}} = \min(\text{upper\_bound}^{\text{fine}}, t_{\text{max}}^{\text{fine}}, t_{\text{max}}^{\text{AABB}})$.
    *   **Sample Generation:**
        *   Generate $N_{\text{coarse}} = \text{num\_samples\_equispaced}$ stratified/uniform samples in $[t_{\text{min}}, t_{\text{max}}]$.
        *   Generate $N_{\text{fine}} = \text{num\_samples\_equispaced}$ stratified/uniform samples in $[t_{\text{min}}^{\text{fine}}, t_{\text{max}}^{\text{fine}}]$.
        *   Concatenate coarse and fine samples.
        *   Sort all samples by distance $t_i$.
        *   Compute positions $\mathbf{p}_i = \mathbf{o} + t_i \mathbf{d}$.

4.  **Rendering (Alpha Computation):**
    *   Compute true cosine $\cos_{\text{true}} = \mathbf{d} \cdot \mathbf{n}$, where $\mathbf{n}$ is the surface normal (gradient of SDF).
    *   Compute cosine annealing ratio $r = \min(1, \text{global\_step} / \text{cos\_anneal\_end})$ if `cos_anneal_end` > 0, else $r=0$.
    *   Compute iterated cosine:
        $$ \cos_{\text{iter}} = -\left(\max(0, -\cos_{\text{true}} \cdot 0.5 + 0.5) \cdot (1 - r) + \max(0, -\cos_{\text{true}}) \cdot r\right) $$
    *   For each sample interval $\Delta t_i = t_{i+1} - t_i$:
        *   Offset SDF: $s_{\text{prev}} = S(\mathbf{p}_i) + \cos_{\text{iter}} \cdot \Delta t_i \cdot 0.5$, $s_{\text{next}} = S(\mathbf{p}_i) - \cos_{\text{iter}} \cdot \Delta t_i \cdot 0.5$.
        *   Compute sigmoid values: $\sigma_{\text{prev}} = \text{sigmoid}(s_{\text{prev}} \cdot \text{inv\_s})$, $\sigma_{\text{next}} = \text{sigmoid}(s_{\text{next}} \cdot \text{inv\_s})$.
        *   Compute alpha: $\alpha_i = \frac{\sigma_{\text{prev}} - \sigma_{\text{next}} + \epsilon}{\sigma_{\text{prev}} + \epsilon}$, where $\epsilon = 10^{-5}$.
        *   Clip $\alpha_i$ to $[0, 1]$.
    *   Composite color and alpha using standard volume rendering equation.

5.  **Variance Modulation (Training Only):**
    *   If `modulate` is active and `global_step` > `mod_start_steps`:
        *   Compute progress $p = \text{global\_step} / \text{reach\_max\_steps}$.
        *   Update $\sigma_{\text{mod\_val}} = \min(\sigma, \sigma_{\text{prev}} + p \cdot (\text{max\_inv\_s} - \sigma_{\text{prev}}))$.
        *   Set effective inverse temperature $\sigma_{\text{eff}} = \min(\sigma, \sigma_{\text{mod\_val}})$.
    *   Else:
        *   $\sigma_{\text{eff}} = \sigma$.
    *   Use $\sigma_{\text{eff}}$ for $\text{inv\_s}$ in alpha computation.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def sigmoid(x):
    return 1.0 / (1.0 + torch.exp(-x))

def compute_alpha(sdf_prev, sdf_next, inv_s, eps=1e-5):
    sig_prev = sigmoid(sdf_prev * inv_s)
    sig_next = sigmoid(sdf_next * inv_s)
    alpha = (sig_prev - sig_next + eps) / (sig_prev + eps)
    return torch.clamp(alpha, 0.0, 1.0)

def render_ray(rays_o, rays_d, sdf_fn, var_net, gs_depth, 
               scene_aabb, radius, num_samples_equispaced, 
               modulate, mod_start_steps, reach_max_steps, max_inv_s,
               global_step, cos_anneal_end, init_val, eps_fd=1e-5):
    
    # 1. Variance Initialization/Modulation
    v = var_net.v # Learnable parameter
    sigma = torch.exp(10.0 * v)
    
    if modulate and global_step > mod_start_steps:
        progress = global_step / reach_max_steps
        # Linear interpolation from previous sigma to max_inv_s
        # Note: In implementation, sigma_prev is usually the current sigma before modulation update
        # Here we assume sigma is the base value, and we modulate the effective inv_s
        sigma_mod_val = torch.min(sigma, sigma + progress * (max_inv_s - sigma))
        inv_s = torch.min(sigma, sigma_mod_val)
    else:
        inv_s = sigma
    
    # Clamp inv_s to prevent explosion/vanishing
    inv_s = torch.clamp(inv_s, 1e-6, 1e6)
    
    # 2. AABB Intersection
    # scene_aabb: [xmin, ymin, zmin, xmax, ymax, zmax]
    # Simplified AABB intersection for ray [o, d]
    t_min_aabb = torch.max((scene_aabb[:3] - rays_o) / rays_d)
    t_max_aabb = torch.min((scene_aabb[3:] - rays_o) / rays_d)
    
    if t_min_aabb >= t_max_aabb:
        return None # Ray misses AABB
    
    # 3. Depth-Guided Sampling Bounds
    if gs_depth is not None:
        # Query SDF at GS depth
        p_depth = rays_o + gs_depth.unsqueeze(1) * rays_d
        s_depth = sdf_fn(p_depth)
        
        # Coarse interval
        interval = torch.abs(s_depth) * radius * 3
        interval = torch.clamp(interval, min=eps_fd * 16)
        
        t_min_coarse = gs_depth - interval
        t_max_coarse = gs_depth + interval
        
        # Fine interval
        interval_fine = torch.abs(s_depth) * radius
        interval_fine = torch.clamp(interval_fine, min=eps_fd * 8)
        
        t_min_fine = gs_depth - interval_fine
        t_max_fine = gs_depth + interval_fine
    else:
        t_min_coarse = t_min_aabb
        t_max_coarse = t_max_aabb
        t_min_fine = t_min_aabb
        t_max_fine = t_max_aabb
        
    # Clamp to AABB
    t_min = torch.max(t_min_coarse, t_min_aabb)
    t_max = torch.min(t_max_coarse, t_max_aabb)
    
    t_min_fine = torch.max(t_min_fine, t_min_aabb)
    t_max_fine = torch.min(t_max_fine, t_max_aabb)
    
    if t_min >= t_max:
        return None
        
    # 4. Sample Generation
    # Stratified sampling
    def stratified_samples(t_min, t_max, num_samples):
        t_min = t_min.unsqueeze(-1)
        t_max = t_max.unsqueeze(-1)
        # Uniform random offsets
        r = torch.rand(num_samples, device=t_min.device)
        # Stratified: divide interval into num_samples bins, pick random in each
        # t_i = t_min + (r_i + i) / N * (t_max - t_min)
        # But standard stratified is: t_i = t_min + (r_i + i) / N * length
        # Here we use the formula from notes: t_i = (r_i + i)/N * (t_max - t_min) + t_min
        # Note: i is 0-indexed or 1-indexed? Notes say r_i + i. Usually i is 0..N-1 or 1..N.
        # Let's assume i is 0..N-1 for 0-indexed, but note says "r_i + i". 
        # Standard stratified: t_i = t_min + (i + r_i) / N * (t_max - t_min)
        indices = torch.arange(num_samples, device=t_min.device)
        t_samples = (r + indices.float()) / num_samples * (t_max - t_min) + t_min
        return t_samples.squeeze(-1)

    t_coarse = stratified_samples(t_min, t_max, num_samples_equispaced)
    t_fine = stratified_samples(t_min_fine, t_max_fine, num_samples_equispaced)
    
    # Concatenate and sort
    t_all = torch.cat([t_coarse, t_fine], dim=-1)
    t_all, sort_idx = torch.sort(t_all, dim=-1)
    
    # Compute distances and positions
    dists = t_all.unsqueeze(-1) # [N_rays, N_samples, 1]
    positions = rays_o.unsqueeze(1) + dists * rays_d.unsqueeze(1) # [N_rays, N_samples, 3]
    
    # 5. Rendering
    # Compute normals for cosine annealing
    # Note: In practice, normals are computed via finite differences or SDF gradient
    # For simplicity, assume sdf_fn returns (sdf, normal) or we compute normal separately
    # Here we assume we have normals n
    # n = compute_normals(sdf_fn, positions) # Placeholder for normal computation
    
    # Cosine annealing
    if cos_anneal_end > 0 and global_step < cos_anneal_end:
        r_anneal = global_step / cos_anneal_end
    else:
        r_anneal = 0.0
        
    # cos_true = d . n
    # cos_iter = -(max(0, -cos_true * 0.5 + 0.5) * (1 - r_anneal) + max(0, -cos_true) * r_anneal)
    # This formula is complex. Let's implement it directly.
    # Note: cos_true is [N_rays, N_samples]
    # cos_iter is [N_rays, N_samples]
    
    # Compute SDF values at samples
    sdf_vals = sdf_fn(positions) # [N_rays, N_samples]
    
    # Compute intervals
    dt = torch.diff(t_all, dim=-1, append=torch.zeros_like(t_all[:, -1:])) # [N_rays, N_samples]
    
    # Compute alpha
    # Offset SDF by +/- iter_cos * dt * 0.5
    # We need normals. Let's assume we have them.
    # If not, this step is incomplete. [G] Assuming normals are available from SDF gradient.
    
    # Placeholder for normal computation
    # n = torch.autograd.grad(sdf_vals, positions, grad_outputs=torch.ones_like(sdf_vals), create_graph=True)[0]
    # n = F.normalize(n, dim=-1)
    # cos_true = torch.sum(rays_d.unsqueeze(1) * n, dim=-1)
    
    # For the sake of the algorithm description, we assume cos_iter is computed
    # cos_iter = ... (see formula)
    
    # SDF offsets
    sdf_prev = sdf_vals + cos_iter * dt * 0.5
    sdf_next = sdf_vals - cos_iter * dt * 0.5
    
    alpha = compute_alpha(sdf_prev, sdf_next, inv_s)
    
    # Composite
    # T_i = product(1 - alpha_j) for j < i
    # C = sum(alpha_i * color_i * T_i)
    # D = sum(alpha_i * t_i * T_i)
    
    return alpha, positions, dists
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of rays and $M$ is the number of samples per ray. Hierarchical sampling reduces the number of empty-space queries, potentially improving effective runtime compared to uniform sampling.
*   **Memory Complexity:** $O(N \cdot M)$ for storing sample positions, distances, and intermediate SDF values. Occupancy grids add $O(R^3)$ memory, where $R$ is the grid resolution (256^3 for foreground, 128^3 for background).
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Reduces floaters and blurry edge artifacts by aligning sampling with predicted depth. Accelerates SDF convergence through mutual guidance with 3DGS.
*   **Limitations:** Requires a 3DGS branch, increasing model complexity and training time. Dependent on the accuracy of GS depth predictions.
*   **When to use:** When high-quality geometry and rendering are required, and a 3DGS branch can be integrated.
*   **When not to use:** When computational resources are limited, or when a simpler SDF-based method suffices.

## Background
The method is part of the GSDF (3DGS meets SDF) framework, introduced by Mulin Yu et al. in 2024 [S1, unofficial]. It builds on Instant-NSR for the SDF branch and Scaffold-GS for the Gaussian branch [S1, unofficial]. The goal is to combine the rendering quality of 3DGS with the geometric reconstruction capabilities of SDFs.

## Sources
[S1] GSDF: 3DGS meets SDF for improved rendering and reconstruction, https://github.com/city-super/GSDF, fetched 2026-10-27 (unofficial)
