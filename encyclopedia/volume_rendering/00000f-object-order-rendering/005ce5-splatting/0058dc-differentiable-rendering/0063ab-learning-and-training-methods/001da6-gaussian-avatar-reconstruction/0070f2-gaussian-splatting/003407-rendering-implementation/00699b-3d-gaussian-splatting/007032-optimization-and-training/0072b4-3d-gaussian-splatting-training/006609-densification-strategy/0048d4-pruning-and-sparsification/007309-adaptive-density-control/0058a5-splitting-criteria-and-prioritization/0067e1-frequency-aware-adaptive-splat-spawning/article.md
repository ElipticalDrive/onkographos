# Frequency-Aware Adaptive Splat Spawning

Also called: Adaptive Frequency-Aware Cell Division, adaptive strategy, frequency-aware spawning

## Overview
Frequency-Aware Adaptive Splat Spawning is a densification strategy that splits existing Gaussians based on a score derived from frequency gaps in the rendered image and alpha holes, performed every 50 steps during the first 85% of training [S1, unofficial]. It spawns new splats in regions of high frequency content or low opacity by sampling from the nearest existing cloud points [S1, unofficial]. Unlike sibling methods that rely primarily on gradient magnitude or view-count density, this method explicitly incorporates spectral frequency analysis to prioritize splitting in visually complex regions [S1, unofficial].

## How it works
**Inputs:**
- Current set of 3D Gaussians (splats) with attributes: means ($\mathbf{p}$), quaternions ($\mathbf{q}$), scales ($\mathbf{s}$), colors ($\mathbf{c}$), opacities ($\alpha$) [S1, unofficial].
- Camera parameters for the current training step: world-to-camera rotation $R_{cam}$, translation $t_{cam}$, and intrinsic matrix $K$ [S1, unofficial].
- Rendered image predictions: predicted color $C_{pred}$ and predicted alpha $\alpha_{pred}$ [S1, unofficial].
- Ground truth image $I_{gt}$ (used to compute frequency gap) [S1, unofficial].
- Training step counter $t$ and total iterations $N_{iterations}$ [S1, unofficial].
- Target splat count $N_{target}$ (default 500,000 if $\le 0$) [S1, unofficial].

**Outputs:**
- Updated set of 3D Gaussians with new splats added via splitting [S1, unofficial].

**Data Structures:**
- Splats are stored as tensors: `means` ($N \times 3$), `quats` ($N \times 4$), `scales` ($N \times 3$), `colors` ($N \times 3$), `opacities` ($N \times 1$) [S1, unofficial].
- Camera matrices: $R_{cam} \in \mathbb{R}^{3 \times 3}$, $t_{cam} \in \mathbb{R}^{3}$, $K \in \mathbb{R}^{3 \times 3}$ [S1, unofficial].

**Steps:**

1. **Validation Check:**
   Proceed only if:
   - $t > 100$
   - $t \pmod{50} == 0$
   - Current splat count $N_{curr} < N_{target}$
   - $t < N_{iterations} \times 85 // 100$ [S1, unofficial].

2. **Compute Frequency Gap:**
   Calculate the frequency gap map $F_{gap}$ between the predicted color $C_{pred}$ and ground truth $I_{gt}$ [S1, unofficial]. The exact implementation of `_compute_freq_gap` is not given in the sources.

3. **Compute Alpha Gap:**
   Calculate the alpha gap map $A_{gap}$ to identify low-opacity regions (holes):
   $$ A_{gap} = \text{ReLU}(0.5 - \alpha_{pred}) $$
   where $\alpha_{pred}$ is the rendered alpha map [S1, unofficial].

4. **Score Splat Contributions:**
   Initialize a score tensor $S$ of size $N_{curr}$ to zeros.
   For each splat $i$:
   - Project splat center $\mathbf{p}_i$ to camera space:
     $$ \mathbf{p}_{cam} = \mathbf{p}_i R_{cam}^T + t_{cam} $$
   - Clamp depth $z = \mathbf{p}_{cam, z}$ to $\min(0.1, z)$ [S1, unofficial].
   - Project to 2D pixel coordinates $(u_i, v_i)$:
     $$ u_i = \left\lfloor \frac{\mathbf{p}_{cam, x}}{z} K_{0,0} + K_{0,2} \right\rfloor $$
     $$ v_i = \left\lfloor \frac{\mathbf{p}_{cam, y}}{z} K_{1,1} + K_{1,2} \right\rfloor $$
   - If $(u_i, v_i)$ are within image bounds and $z > 0.1$:
     $$ S_i = F_{gap}(u_i, v_i) + 2.0 \times A_{gap}(u_i, v_i) $$
   - Else, $S_i = 0$ [S1, unofficial].

5. **Select Splats to Split:**
   Calculate the number of splats to split:
   $$ N_{split} = \min\left( \left\lfloor \frac{N_{curr}}{10} \right\rfloor, N_{target} - N_{curr}, 5000 \right) $$
   Select the indices of the top $N_{split}$ splats with the highest scores using `torch.topk` [S1, unofficial].

6. **Split Selected Splats:**
   For each selected parent splat $j$:
   - Identify the longest axis of the scale:
     $$ a_{long} = \arg\max_k (\mathbf{s}_j)_k $$
   - Compute the world-space direction of the longest axis:
     $$ \mathbf{d}_{world} = R_{splat}[j, :, a_{long}] $$
     where $R_{splat}$ is the rotation matrix derived from $\mathbf{q}_j$ [S1, unofficial].
   - Compute the offset distance:
     $$ d_{offset} = \exp((\mathbf{s}_j)_{a_{long}}) \times 0.5 $$
   - Compute the offset vector:
     $$ \mathbf{v}_{offset} = \mathbf{d}_{world} \times d_{offset} $$
   - Create two child splats (parent and child) for all attributes:
     - **Means:**
       $$ \mathbf{p}_{parent} = \mathbf{p}_j + \mathbf{v}_{offset} $$
       $$ \mathbf{p}_{child} = \mathbf{p}_j - \mathbf{v}_{offset} $$
     - **Scales:**
       Halve the longest axis in log space:
       $$ \mathbf{s}_{new} = \mathbf{s}_j $$
       $$ (\mathbf{s}_{new})_{a_{long}} = (\mathbf{s}_j)_{a_{long}} - \ln(2) \approx (\mathbf{s}_j)_{a_{long}} - 0.693 $$
       Both parent and child use $\mathbf{s}_{new}$ [S1, unofficial].
     - **Other Attributes:**
       Clone colors, quaternions, and opacities from the parent [S1, unofficial].

7. **Update Cloud:**
   Concatenate the new parent and child splats along dimension 0 for all parameters, replacing the original parent splats [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def frequency_aware_spawn(splats, camera_params, rendered_pred, gt_image, step, n_iterations, target_splats=500000):
    """
    splats: dict with keys 'means', 'quats', 'scales', 'colors', 'opacities' (tensors of shape N, ...)
    camera_params: dict with 'R_cam' (3,3), 't_cam' (3,), 'K' (3,3)
    rendered_pred: dict with 'colors' (H, W, 3), 'alpha' (H, W, 1)
    gt_image: tensor (H, W, 3)
    step: int
    n_iterations: int
    target_splats: int
    """
    # 1. Validation Check
    if step <= 100 or step % 50 != 0 or step >= n_iterations * 85 // 100:
        return splats
    
    n_curr = splats['means'].shape[0]
    if n_curr >= target_splats:
        return splats

    # 2. Compute Frequency Gap
    # Not given in the sources.
    freq_gap = torch.zeros_like(rendered_pred['colors'][..., 0])
    
    # 3. Compute Alpha Gap
    alpha_pred = rendered_pred['alpha'].squeeze(-1) # (H, W)
    alpha_gap = torch.relu(0.5 - alpha_pred)
    
    # 4. Score Splat Contributions
    means = splats['means'] # (N, 3)
    R_cam = camera_params['R_cam']
    t_cam = camera_params['t_cam']
    K = camera_params['K']
    
    # Project to camera space
    means_cam = means @ R_cam.T + t_cam # (N, 3)
    z = means_cam[:, 2]
    z = torch.clamp(z, min=0.1)
    
    # Project to 2D
    u = ((means_cam[:, 0] / z) * K[0, 0] + K[0, 2]).long()
    v = ((means_cam[:, 1] / z) * K[1, 1] + K[1, 2]).long()
    
    H, W = alpha_gap.shape
    valid_mask = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z > 0.1)
    
    scores = torch.zeros(n_curr, device=means.device)
    scores[valid_mask] = freq_gap[v[valid_mask], u[valid_mask]] + 2.0 * alpha_gap[v[valid_mask], u[valid_mask]]
    
    # 5. Select Splats to Split
    n_split = min(n_curr // 10, target_splats - n_curr, 5000)
    if n_split <= 0:
        return splats
        
    _, top_indices = torch.topk(scores, n_split)
    parent_indices = top_indices
    
    # 6. Split Selected Splats
    parent_means = means[parent_indices]
    parent_quats = splats['quats'][parent_indices]
    parent_scales = splats['scales'][parent_indices]
    parent_colors = splats['colors'][parent_indices]
    parent_opacities = splats['opacities'][parent_indices]
    
    # Compute rotation matrices from quaternions
    R_splat = _quat_to_rotmat(parent_quats) # (n_split, 3, 3)
    
    # Find longest axis
    longest_ax = torch.argmax(parent_scales, dim=-1) # (n_split,)
    
    # Get direction vectors
    ax_dir = R_splat[torch.arange(n_split), :, longest_ax] # (n_split, 3)
    
    # Compute offset
    scale_long = parent_scales[torch.arange(n_split), longest_ax] # (n_split,)
    offset_dist = torch.exp(scale_long) * 0.5
    offset = ax_dir * offset_dist.unsqueeze(-1) # (n_split, 3)
    
    # New means
    new_means_parent = parent_means + offset
    new_means_child = parent_means - offset
    
    # New scales (halve longest axis in log space)
    new_scales = parent_scales.clone()
    new_scales[torch.arange(n_split), longest_ax] -= 0.693 # log(2)
    
    # Concatenate
    new_means = torch.cat([new_means_parent, new_means_child], dim=0)
    new_quats = torch.cat([parent_quats, parent_quats], dim=0)
    new_scales = torch.cat([new_scales, new_scales], dim=0)
    new_colors = torch.cat([parent_colors, parent_colors], dim=0)
    new_opacities = torch.cat([parent_opacities, parent_opacities], dim=0)
    
    # Remove old parents and add new children
    mask = torch.ones(n_curr, dtype=torch.bool, device=means.device)
    mask[parent_indices] = False
    kept_means = means[mask]
    kept_quats = splats['quats'][mask]
    kept_scales = splats['scales'][mask]
    kept_colors = splats['colors'][mask]
    kept_opacities = splats['opacities'][mask]
    
    final_means = torch.cat([kept_means, new_means], dim=0)
    final_quats = torch.cat([kept_quats, new_quats], dim=0)
    final_scales = torch.cat([kept_scales, new_scales], dim=0)
    final_colors = torch.cat([kept_colors, new_colors], dim=0)
    final_opacities = torch.cat([kept_opacities, new_opacities], dim=0)
    
    splats['means'] = final_means
    splats['quats'] = final_quats
    splats['scales'] = final_scales
    splats['colors'] = final_colors
    splats['opacities'] = final_opacities
    
    return splats

def _quat_to_rotmat(q):
    """Convert quaternion (w, x, y, z) to rotation matrix."""
    w, x, y, z = q[:, 0], q[:, 1], q[:, 2], q[:, 3]
    N = w*w + x*x + y*y + z*z
    w, x, y, z = w/N, x/N, y/N, z/N
    
    r = torch.zeros(q.shape[0], 3, 3, device=q.device)
    r[:, 0, 0] = 1 - 2 * (y*y + z*z)
    r[:, 0, 1] = 2 * (x*y - w*z)
    r[:, 0, 2] = 2 * (x*z + w*y)
    r[:, 1, 0] = 2 * (x*y + w*z)
    r[:, 1, 1] = 1 - 2 * (x*x + z*z)
    r[:, 1, 2] = 2 * (y*z - w*x)
    r[:, 2, 0] = 2 * (x*z - w*y)
    r[:, 2, 1] = 2 * (y*z + w*x)
    r[:, 2, 2] = 1 - 2 * (x*x + y*y)
    return r
```

**Parameters:**
- `target_splats`: Target number of splats. If $\le 0$, defaults to 500,000 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Depth clamping to 0.1 prevents division by zero or extreme projection artifacts for splats behind the camera [S1, unofficial].
- **Boundaries:** Pixel coordinates must be checked against image dimensions $(H, W)$ to avoid out-of-bounds access to frequency and alpha maps [G].
- **Degenerate Input:** If no splats have positive scores, `n_split` may be 0, and no splitting occurs [S1, unofficial].
- **Frequency Gap Implementation:** The source does not specify how `_compute_freq_gap` is computed [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per step for scoring and splitting, where $N$ is the number of splats [S1, unofficial].
- **Memory Complexity:** Increases by $N_{split}$ splats per step [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of splats [S1, unofficial].
- **Strengths:** Explicitly targets high-frequency details and holes, leading to better reconstruction of complex textures and thin structures [S1, unofficial].
- **Limitations:** Requires computation of frequency gap, which may be non-trivial to implement efficiently [S1, unofficial].
- **When to Use:** When reconstruction quality in high-frequency regions is prioritized over uniform density [S1, unofficial].
- **When Not to Use:** When computational resources are limited and frequency analysis is too expensive, or when the scene is low-frequency [S1, unofficial].

## Background
Originating from the `dust3r-3dgs` repository, this method is an adaptation of densification strategies for splat-based rendering, specifically designed to work with scenes reconstructed from DUSt3R/MASt3R outputs [S1, unofficial]. It was developed to address the limitations of standard gradient-based splitting by incorporating frequency-aware metrics [S1, unofficial].

## Sources
[S1] SpaceCollectors/dust3r-3dgs, https://github.com/SpaceCollectors/dust3r-3dgs, fetched 2026-10-27 (unofficial)
