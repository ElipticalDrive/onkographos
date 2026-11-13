# Hard Sample Mining

Also called: HSM

## Overview
Hard Sample Mining (HSM) is an optimization procedure that reduces the computational cost of NeRF training by subsampling a "hard minibatch" of point samples based on their propagated pixel loss, rather than using all samples or pruning based on density weights [S1]. It differs from sibling methods like [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md) which focus on spatial anti-aliasing, or [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) which uses hierarchical density estimation, by directly ranking samples according to the gradient signal they provide to the network parameters [S1]. Unlike [Nerfacc](9146-two-stage-sampling-with-occupancy-grid-and-nonlinear-functio.md) which prunes low-weight samples, HSM retains high-loss samples even if their density weight is low, ensuring that difficult-to-learn regions drive the optimization [S1].

## How it works
**Inputs:**
*   A batch of rays, each with $N$ sampled points $x_i$ along the ray.
*   The rendered color $C$ and the ground truth color $C_{gt}$ for the batch.
*   The total batch size parameter $B$ (typical values: 220 for Instant-NGP, 218 for Nerfacto) [S1].

**Outputs:**
*   A subset of point samples (the hard minibatch) used to compute the loss and update the network.

**Data Structures:**
*   **Point Loss Array:** A 1D array of size equal to the total number of point samples in the batch, storing the per-point contribution to the pixel loss.
*   **Hard Minibatch Indices:** A list of indices corresponding to the selected points.

**Steps:**
1.  **Forward Pass:** Render the batch of rays using the current NeRF model to obtain predicted colors $C$.
2.  **Loss Computation:** Compute the pixel-wise loss (e.g., $L_2$ or $L_1$) between $C$ and $C_{gt}$.
3.  **Point-Level Loss Propagation:** Distribute the pixel loss to the individual point samples along each ray. The exact distribution mechanism depends on the rendering equation (e.g., alpha compositing weights), but the goal is to assign a scalar loss value $l_i$ to each point $x_i$ that reflects its contribution to the total error [S1].
4.  **Ranking:** Sort all point samples in the batch in descending order of their point-level loss $l_i$ [S1].
5.  **Subsampling:** Select the top $K$ points with the highest loss, where $K$ is determined by the target batch size $B$ and the average number of points per ray. This subset forms the "hard minibatch" [S1].
6.  **Backward Pass:** Compute gradients and update network parameters using only the hard minibatch [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_sample_mining(rays, gt_colors, nerf_model, batch_size_B):
    """
    Performs Hard Sample Mining for NeRF training.
    
    Args:
        rays: List of rays, each containing N sample points.
        gt_colors: Ground truth colors for the rays.
        nerf_model: The NeRF network.
        batch_size_B: Target number of point samples for the hard minibatch.
        
    Returns:
        Updated network parameters.
    """
    # 1. Forward pass to get rendered colors
    # Assume render_rays returns predicted colors C for all rays
    C = render_rays(rays, nerf_model)
    
    # 2. Compute pixel loss
    # Using L2 loss as an example; other losses may be used
    pixel_losses = torch.mean((C - gt_colors) ** 2, dim=-1)
    
    # 3. Propagate pixel loss to point samples
    # This step requires access to the alpha compositing weights or the specific
    # loss gradient with respect to each point's contribution.
    # Let point_losses be a tensor of shape [num_rays, num_points_per_ray]
    # The exact formula depends on the rendering differentiability.
    # A common approximation is weighting the pixel loss by the transmittance/alpha.
    # [G] Assuming a standard alpha compositing model, the gradient w.r.t. point color
    # is proportional to the accumulated transmittance T_i.
    # point_losses = pixel_losses.unsqueeze(-1) * transmittance_weights
    
    # For the purpose of this implementation, we assume a helper function
    # that computes the per-point loss contribution.
    point_losses = compute_point_level_losses(rays, C, gt_colors, nerf_model)
    
    # 4. Flatten and rank
    # Flatten all point losses into a single array
    flat_losses = point_losses.flatten()
    
    # Get indices of the top K hardest samples
    # K is typically set to batch_size_B or a fraction thereof
    # If the total points < B, use all points
    num_points = flat_losses.shape[0]
    k = min(batch_size_B, num_points)
    
    # Get indices of the top k losses
    _, top_indices = torch.topk(flat_losses, k)
    
    # 5. Map flat indices back to ray and point indices
    # [G] Implementation detail: Convert flat indices to (ray_idx, point_idx)
    ray_indices = top_indices // num_points_per_ray
    point_indices = top_indices % num_points_per_ray
    
    # 6. Backward pass on the hard minibatch
    # Re-render or compute gradients only for the selected points
    # This step is non-trivial in standard autograd frameworks if not carefully structured.
    # Typically, one masks out the gradients for non-selected points.
    # [G] Assuming a framework that supports selective gradient computation:
    mask = torch.zeros_like(flat_losses)
    mask[top_indices] = 1.0
    # Reshape mask to match point_losses shape
    mask = mask.reshape(point_losses.shape)
    
    # Compute loss only for selected points
    hard_loss = (point_losses * mask).sum()
    
    # Backward pass
    hard_loss.backward()
    
    return nerf_model
```

**Parameters:**
*   $B$: Total batch size for the hard minibatch. Typical values are 220 for Instant-NGP and 218 for Nerfacto [S1].

**Edge Cases and Pitfalls:**
*   **Uniformly Challenging Content:** In scenes with unbounded backgrounds or multiple outward-facing objects, the pixel loss distribution becomes more uniform, limiting the computational efficiency gains of HSM [S1].
*   **Pose Errors:** Severe pose mistakes in the dataset can create "impossible to learn" point samples that consistently rank as hard, crowding the hard minibatch and potentially destabilizing training [S1]. Techniques like uncertainty or pose estimation may be needed to mitigate this [S1].

## Operating characteristics
*   **Time Complexity:** Reduces iteration time by focusing gradient updates on a smaller subset of samples [S1].
*   **Memory Complexity:** Reduces dynamic memory footprint. For example, in Nerfacto, memory usage dropped from 300 MB to 160 MB for normal iterations [S1].
*   **Performance:** On Mip-NeRF-360 scenes, HSM converges ~2x faster to a given PSNR level and achieves ~1 dB higher validation PSNR per same training time compared to the baseline [S1].
*   **Strengths:** Effective for methods like Instant-NGP where grid sampling is used, as it focuses computation on informative regions [S1].
*   **Limitations:** Less effective in scenes with uniform difficulty; less impactful when proposal networks already prune low-weight samples effectively (e.g., in Nerfacto) [S1].

## Background
Hard Sample Mining was introduced by J. Korhonen et al. in the paper "Efficient NeRF Optimization -- Not All Samples Remain Equally Hard" [S1]. The method was applied to PyTorch reimplementations of Instant-NGP and Nerfacto on Mip-NeRF-360 and synthetic NeRF datasets [S1]. It is distinct from prior importance sampling techniques by explicitly using propagated pixel loss to rank samples rather than density or transmittance weights [S1].

## Sources
[S1] Efficient NeRF Optimization -- Not All Samples Remain Equally Hard, https://arxiv.org/abs/2408.03193v1, fetched 2026-10-27
