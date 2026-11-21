# Frequency-Aware Adaptive Splat Spawning

Also called: Adaptive Frequency-Aware Cell Division, adaptive strategy, frequency-aware spawning

## Overview
Frequency-Aware Adaptive Splat Spawning is a densification strategy for 3D Gaussian Splatting that spawns new splats in regions of high-frequency content or low opacity (holes) by scoring pixels based on frequency gaps and alpha gaps, then sampling new splats from the nearest existing cloud points [S1, unofficial]. It differs from sibling methods by explicitly incorporating image frequency analysis into the splitting decision, targeting detail loss rather than just gradient magnitude or opacity thresholds [S1, unofficial].

## How it works
**Inputs:**
- Rendered color image $I_{pred}$ and alpha map $\alpha_{pred}$ for the current training step.
- Current set of 3D Gaussians (splats) with attributes: means $\mu$, scales $s$, rotations $q$, colors $c$, opacities $\alpha$.
- Camera extrinsics $W2C$ (rotation $R$, translation $t$) and intrinsics $K$ ($f_x, f_y, c_x, c_y$).
- Current training step $t_{step}$ and total iterations $T_{total}$.
- Target splat count $N_{target}$ (default 500,000 if $\le 0$) [S1, unofficial].

**Outputs:**
- Updated set of 3D Gaussians with new splats added via splitting existing ones.

**Data Structures:**
- Splats: List of objects or tensors containing $\mu \in \mathbb{R}^3$, $s \in \mathbb{R}^3$, $q \in \mathbb{R}^4$, $c \in \mathbb{R}^3$, $\alpha \in \mathbb{R}$.
- Score Map: 2D tensor of size $H \times W$ storing spawn scores per pixel.

**Steps:**

1. **Check Spawning Conditions:**
   Proceed only if:
   - $t_{step} > 100$
   - $t_{step} \pmod{50} == 0$
   - Current splat count $N_{curr} < N_{target}$
   - $t_{step} < 0.85 \times T_{total}$ [S1, unofficial]

2. **Compute Spawn Scores:**
   - Initialize `splat_scores` tensor of zeros, size matching the number of splats.
   - Project all splat centers to 2D pixel coordinates $(u, v)$ in camera space:
     $$ \mu_{cam} = R \mu + t $$
     $$ z = \mu_{cam, z} $$
     $$ u = \left\lfloor \frac{\mu_{cam, x}}{z} f_x + c_x \right\rfloor $$
     $$ v = \left\lfloor \frac{\mu_{cam, y}}{z} f_y + c_y \right\rfloor $$
     Clamp $z \ge 0.1$ [S1, unofficial].
   - Filter valid splats where $u, v$ are within image bounds and $z > 0.1$.
   - Compute **Frequency Gap**:
     $$ \text{freq\_gap} = \text{ComputeFrequencyGap}(I_{pred}, I_{gt}) $$
     *Note: The exact formula for `ComputeFrequencyGap` is not provided in the source, but it measures high-frequency discrepancies between predicted and ground truth images [S1, unofficial].*
   - Compute **Alpha Gap**:
     $$ \text{alpha\_gap} = \max(0, 0.5 - \alpha_{pred}) $$
     *Note: $\alpha_{pred}$ is the rendered alpha map [S1, unofficial].*
   - Assign scores to valid splats based on their projected pixel coordinates:
     $$ \text{splat\_scores}[i] = \text{freq\_gap}[u_i, v_i] + 2.0 \times \text{alpha\_gap}[u_i, v_i] $$
     [S1, unofficial]

3. **Select Splats to Split:**
   - Calculate number of splats to split:
     $$ N_{split} = \min\left(\left\lfloor \frac{N_{curr}}{10} \right\rfloor, N_{target} - N_{curr}, 5000\right) $$
     [S1, unofficial]
   - Select top $N_{split}$ splats with highest scores using `torch.topk`. Let these indices be $I_{split}$ [S1, unofficial].

4. **Split Selected Splats:**
   For each parent splat $p \in I_{split}$:
   - Find the longest axis of the Gaussian:
     $$ a_{long} = \arg\max_j(s_p[j]) $$
     [S1, unofficial]
   - Compute the world-space direction of the longest axis:
     $$ d_{world} = R_p[:, a_{long}] $$
     where $R_p$ is the rotation matrix derived from quaternion $q_p$ [S1, unofficial].
   - Compute offset distance:
     $$ d_{offset} = \exp(s_p[a_{long}]) \times 0.5 $$
     [S1, unofficial]
   - Compute offset vector:
     $$ \Delta = d_{world} \times d_{offset} $$
     [S1, unofficial]
   - Create two child splats (parent and child):
     - **Means**:
       $$ \mu_{parent} = \mu_p + \Delta $$
       $$ \mu_{child} = \mu_p - \Delta $$
       [S1, unofficial]
     - **Scales**: Halve the longest axis in log space:
       $$ s'_{parent}[j] = s'_{child}[j] = s_p[j] \quad \text{for } j \neq a_{long} $$
       $$ s'_{parent}[a_{long}] = s'_{child}[a_{long}] = s_p[a_{long}] - \ln(2) $$
       *Note: The source subtracts 0.693 ($\ln 2$) from the scale value, implying scales are stored in log space or the operation is applied to log-scales. If scales are linear, this operation is invalid. Given the context of "subtracting 0.693", it is assumed scales are in log space or the source implies log-scale representation [S1, unofficial].*
     - **Other Attributes**: Copy $q, c, \alpha$ from parent to both children.
   - Replace the parent splat with the two children in the splat list [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def frequency_aware_spawn(splats, images_pred, images_gt, alphas_pred, 
                          w2c, intrinsic, step, total_iters, target_splats=500000):
    """
    Perform frequency-aware adaptive splat spawning.
    
    Args:
        splats: Dict of tensors for splat attributes (means, scales, quats, colors, opacities).
        images_pred: Rendered color images (B, H, W, 3).
        images_gt: Ground truth color images (B, H, W, 3).
        alphas_pred: Rendered alpha maps (B, H, W, 1).
        w2c: World-to-camera transformation (B, 3, 4) [R|t].
        intrinsic: Camera intrinsic matrix (B, 3, 3).
        step: Current training step.
        total_iters: Total training iterations.
        target_splats: Target number of splats.
        
    Returns:
        Updated splats dict.
    """
    # 1. Check conditions
    if step <= 100 or step % 50 != 0 or step >= 0.85 * total_iters:
        return splats
    
    n_curr = splats['means'].shape[0]
    if n_curr >= target_splats:
        return splats
    
    # 2. Compute scores
    # Project splat centers to 2D
    means = splats['means']  # (N, 3)
    R_cam = w2c[:, :3, :3]   # (B, 3, 3)
    t_cam = w2c[:, :3, 3]    # (B, 3)
    
    # Assuming single camera for simplicity, or batched. 
    # The source implies a single view or per-view processing. 
    # We'll assume batch dimension B=1 for the main logic, or loop over B.
    # For clarity, we assume single camera parameters for the spawn logic 
    # as the source code snippet uses single extrinsic/intrinsic.
    
    # Transform means to camera space
    means_cam = means @ R_cam[0].T + t_cam[0]  # (N, 3)
    z = means_cam[:, 2]
    
    # Clamp z
    z = torch.clamp(z, min=0.1)
    
    # Project to 2D
    fx, fy = intrinsic[0, 0, 0], intrinsic[0, 1, 1]
    cx, cy = intrinsic[0, 0, 2], intrinsic[0, 1, 2]
    
    u = ((means_cam[:, 0] / z) * fx + cx).long()
    v = ((means_cam[:, 1] / z) * fy + cy).long()
    
    # Filter valid splats
    H, W = images_pred.shape[1:3]
    valid_mask = (u >= 0) & (u < W) & (v >= 0) & (v < H) & (z > 0.1)
    
    if not torch.any(valid_mask):
        return splats
        
    # Compute frequency gap and alpha gap
    # Note: ComputeFrequencyGap is not defined in source. 
    # We assume it returns a tensor of same shape as images_pred[...0]
    freq_gap = compute_frequency_gap(images_pred, images_gt) # Placeholder for undefined function
    alpha_gap = torch.relu(0.5 - alphas_pred.squeeze(-1)) # (B, H, W)
    
    # Initialize scores
    splat_scores = torch.zeros(n_curr, device=means.device)
    
    # Assign scores to valid splats
    # Flatten indices for indexing
    valid_u = u[valid_mask]
    valid_v = v[valid_mask]
    
    # Accumulate scores (if multiple splats project to same pixel, sum? Source says "set score")
    # Source: "set score to freq_gap at projected pixel coordinates"
    # This implies a map-based accumulation or direct assignment. 
    # We'll use scatter_add to handle overlaps if any, or direct assignment if unique.
    # Given the source uses "set", we assume direct assignment or max. 
    # Let's use scatter_add for robustness.
    
    freq_gap_val = freq_gap[0, valid_v, valid_u] # (N_valid,)
    alpha_gap_val = alpha_gap[0, valid_v, valid_u] # (N_valid,)
    
    scores = freq_gap_val + 2.0 * alpha_gap_val
    
    # Scatter scores to splat_scores
    splat_scores[valid_mask] = scores
    
    # 3. Select splats to split
    n_split = min(n_curr // 10, target_splats - n_curr, 5000)
    if n_split <= 0:
        return splats
        
    _, topk_idx = torch.topk(splat_scores, n_split)
    split_idx = topk_idx[splat_scores[topk_idx] > 0] # Ensure positive scores
    
    if split_idx.numel() == 0:
        return splats
        
    # 4. Split splats
    parent_means = means[split_idx]
    parent_scales = splats['scales'][split_idx]
    parent_quats = splats['quats'][split_idx]
    parent_colors = splats['colors'][split_idx]
    parent_opacities = splats['opacities'][split_idx]
    
    # Find longest axis
    longest_ax = torch.argmax(parent_scales, dim=-1) # (N_split,)
    
    # Get rotation matrices
    R_splat = quat_to_rotmat(parent_quats) # (N_split, 3, 3)
    
    # Direction of longest axis
    ax_dir = R_splat[torch.arange(len(split_idx)), :, longest_ax] # (N_split, 3)
    
    # Offset distance
    offset_dist = torch.exp(parent_scales[torch.arange(len(split_idx)), longest_ax]) * 0.5
    
    # Offset vector
    offset = ax_dir * offset_dist.unsqueeze(-1) # (N_split, 3)
    
    # New means
    new_means_parent = parent_means + offset
    new_means_child = parent_means - offset
    
    # New scales (halve longest axis in log space)
    # Assuming scales are in log space as per source subtraction of ln(2)
    new_scales = parent_scales.clone()
    new_scales[torch.arange(len(split_idx)), longest_ax] -= np.log(2)
    
    # Concatenate
    new_means = torch.cat([new_means_parent, new_means_child], dim=0)
    new_scales = torch.cat([new_scales, new_scales], dim=0)
    new_quats = torch.cat([parent_quats, parent_quats], dim=0)
    new_colors = torch.cat([parent_colors, parent_colors], dim=0)
    new_opacities = torch.cat([parent_opacities, parent_opacities], dim=0)
    
    # Remove old splats and add new ones
    mask = torch.ones(n_curr, dtype=torch.bool)
    mask[split_idx] = False
    remaining_means = means[mask]
    remaining_scales = splats['scales'][mask]
    remaining_quats = splats['quats'][mask]
    remaining_colors = splats['colors'][mask]
    remaining_opacities = splats['opacities'][mask]
    
    final_means = torch.cat([remaining_means, new_means], dim=0)
    final_scales = torch.cat([remaining_scales, new_scales], dim=0)
    final_quats = torch.cat([remaining_quats, new_quats], dim=0)
    final_colors = torch.cat([remaining_colors, new_colors], dim=0)
    final_opacities = torch.cat([remaining_opacities, new_opacities], dim=0)
    
    splats['means'] = final_means
    splats['scales'] = final_scales
    splats['quats'] = final_quats
    splats['colors'] = final_colors
    splats['opacities'] = final_opacities
    
    return splats

def compute_frequency_gap(pred, gt):
    """Placeholder for undefined frequency gap computation."""
    # Source does not define this function. 
    # It likely involves FFT or Laplacian pyramid differences.
    return torch.abs(pred - gt) # Dummy implementation
```

**Edge Cases and Pitfalls:**
- **Frequency Gap Undefined:** The source does not provide the implementation for `ComputeFrequencyGap`. Implementers must define this, e.g., using high-pass filtering or Laplacian differences [S1, unofficial].
- **Scale Representation:** The source subtracts $\ln(2)$ from scales. This implies scales are stored in log space. If scales are linear, this operation is incorrect. Implementers must ensure consistency [S1, unofficial].
- **Pixel Overlap:** Multiple splats may project to the same pixel. The source uses "set score", which may imply overwriting. Using `scatter_add` or `scatter_max` is safer to accumulate scores [S1, unofficial].
- **Boundary Conditions:** Splats projecting outside image bounds or with $z \le 0.1$ are ignored [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ for projection and score computation, where $N$ is the number of splats. Splitting is $O(N_{split})$. Overall per-step cost is linear in $N$.
- **Memory Complexity:** $O(N)$ for storing splat attributes and score maps.
- **Performance:** Effective at filling holes and adding detail in high-frequency regions. May increase training time due to frequent splitting [S1, unofficial].
- **Strengths:** Targets both geometric detail (frequency) and coverage (alpha holes).
- **Limitations:** Depends on the quality of the frequency gap metric. May over-split in noisy regions if frequency gap is not robust [S1, unofficial].

## Background
Frequency-Aware Adaptive Splat Spawning is an adaptation of densification strategies for 3D Gaussian Splatting, specifically designed to address limitations in standard gradient-based splitting by incorporating image frequency information. It is used in the context of Gaussian Avatar Reconstruction and differentiable rendering pipelines [S1, unofficial].

## Sources
[S1] SpaceCollectors/dust3r-3dgs, https://github.com/SpaceCollectors/dust3r-3dgs, fetched 2026-10-27 (unofficial)
