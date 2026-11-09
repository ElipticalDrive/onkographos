# Hard Sample Mining

Also called: HSM

## Overview
Hard Sample Mining (HSM) is an optimization procedure that replaces uniform or density-based ray sampling with a strategy that ranks point samples by their propagated pixel loss and subsamples a "hard" minibatch for gradient updates. It differs from sibling methods like [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md) or [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by explicitly using the scalar error signal (pixel loss) to determine sample importance, rather than relying solely on geometric priors like density or transmittance. This allows the optimizer to focus computational resources on regions where the model is currently performing poorly, accelerating convergence and reducing memory usage.

## How it works
**Inputs and Outputs:**
*   **Input:** A batch of rays, each with a set of sampled points $\{p_i\}$, their associated densities $\sigma_i$, transmittances $T_i$, and colors $c_i$. The current network parameters $\theta$. The target image $I_{target}$ and the rendered image $I_{render}$.
*   **Output:** A subset of points (the hard minibatch) used to compute the gradient update for $\theta$.

**Data Structures:**
*   **Point Samples:** A list or tensor of shape $(N, D)$, where $N$ is the total number of samples across the batch and $D$ is the feature dimension (position, density, color, etc.).
*   **Loss Tensor:** A tensor of shape $(N, 1)$ containing the per-sample contribution to the pixel loss.
*   **Hard Minibatch Indices:** A list of indices into the original point samples, selecting the top $K$ hardest samples.

**Steps:**
1.  **Forward Pass:** Render the current batch of rays using the current network parameters $\theta$ to obtain $I_{render}$.
2.  **Loss Computation:** Compute the pixel-wise loss $\mathcal{L}_{pixel} = \| I_{render} - I_{target} \|^2$ (or other metric).
3.  **Loss Propagation to Samples:** Assign a loss value $l_i$ to each sample point $p_i$ along each ray. The loss for a sample is typically proportional to its weight $w_i$ in the volume rendering equation and the residual error at its pixel location.
    $$ w_i = T_i (1 - e^{-\sigma_i \delta_i}) $$
    $$ T_i = e^{-\sum_{j < i} \sigma_j \delta_j} $$
    $$ l_i = w_i \cdot \| c_i - I_{target}(u, v) \|^2 $$
    where $\delta_i$ is the distance between sample $i$ and $i+1$, and $(u, v)$ are the pixel coordinates. [G] If the source does not specify the exact propagation formula, the standard approach is to distribute the pixel loss proportionally to the sample weights $w_i$.
4.  **Ranking:** Rank all $N$ samples in the batch by their assigned loss $l_i$ in descending order.
5.  **Subsampling:** Select the top $K$ samples with the highest loss to form the hard minibatch. The size of the hard minibatch is determined by the total batch size $B$ and a subsampling ratio.
    $$ K = \lfloor B \cdot \alpha \rfloor $$
    where $\alpha$ is the subsampling ratio (e.g., 0.5). [G] The exact value of $\alpha$ is not specified in the sources; it is a hyperparameter.
6.  **Gradient Update:** Compute the gradient of the loss with respect to $\theta$ using only the hard minibatch.
    $$ \nabla_\theta \mathcal{L} = \frac{1}{K} \sum_{i \in \text{Hard Minibatch}} \nabla_\theta l_i $$
7.  **Parameter Update:** Update $\theta$ using an optimizer (e.g., Adam).

**Parameters:**
*   **Batch Size ($B$):** Total number of rays processed per iteration. Typical values: 220 for Instant-NGP, 218 for Nerfacto. [S1]
*   **Subsampling Ratio ($\alpha$):** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Uniformly Challenging Content:** In scenes with unbounded backgrounds or multiple outward-facing objects, the pixel loss distribution is more uniform, limiting the efficiency gains of HSM. [S1]
*   **Pose Errors:** Severe pose mistakes in the dataset can produce "impossible to learn" point samples that consistently rank as hard, crowding the hard minibatch and destabilizing training. [S1] Mitigation techniques like uncertainty or pose estimation can be applied. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_sample_mining(rays, network, target_images, batch_size, subsample_ratio):
    """
    rays: List of Ray objects, each containing positions, directions, and pixel coordinates.
    network: The NeRF network model.
    target_images: Tensor of target images for the rays in the batch.
    batch_size: Total number of rays to process.
    subsample_ratio: Fraction of samples to keep (e.g., 0.5).
    """
    # 1. Forward Pass
    # Render rays and get per-sample outputs
    # outputs: dict with 'sigma', 'color', 'transmittance', etc.
    # shape: (num_samples, feature_dim)
    outputs = network(rays)
    
    # 2. Loss Computation
    # Compute rendered image from samples
    rendered_images = volume_rendering(outputs['sigma'], outputs['color'], outputs['transmittance'])
    
    # Compute pixel-wise loss
    # loss_per_pixel: (batch_size,)
    loss_per_pixel = torch.mean((rendered_images - target_images) ** 2, dim=(1, 2))
    
    # 3. Loss Propagation to Samples
    # Distribute pixel loss to samples based on weights
    # weights: (num_samples,)
    weights = outputs['transmittance'] * (1 - torch.exp(-outputs['sigma'] * outputs['delta']))
    
    # Normalize weights to sum to 1 per ray, then multiply by pixel loss
    # This is a simplified propagation; exact formula may vary
    # loss_per_sample: (num_samples,)
    loss_per_sample = torch.zeros_like(weights)
    for i, ray in enumerate(rays):
        sample_indices = ray.sample_indices
        ray_loss = loss_per_pixel[i]
        ray_weights = weights[sample_indices]
        ray_weights_sum = ray_weights.sum()
        if ray_weights_sum > 0:
            loss_per_sample[sample_indices] = ray_weights / ray_weights_sum * ray_loss
        else:
            loss_per_sample[sample_indices] = 0
            
    # 4. Ranking
    # Get indices of top K samples
    num_samples = loss_per_sample.shape[0]
    num_hard_samples = int(num_samples * subsample_ratio)
    
    # Get indices of the hardest samples
    hard_indices = torch.topk(loss_per_sample, num_hard_samples).indices
    
    # 5. Gradient Update
    # Compute gradients using only hard samples
    # Re-render or use cached values for hard samples
    hard_outputs = {k: v[hard_indices] for k, v in outputs.items()}
    hard_target_images = target_images[hard_indices // samples_per_ray] # Adjust for ray grouping
    
    # Compute loss for hard samples
    hard_rendered = volume_rendering(hard_outputs['sigma'], hard_outputs['color'], hard_outputs['transmittance'])
    hard_loss = torch.mean((hard_rendered - hard_target_images) ** 2)
    
    # Backward pass
    hard_loss.backward()
    
    # 6. Parameter Update
    optimizer.step()
    optimizer.zero_grad()
```

## Subtopics

### Loss Propagation Strategies
Methods for distributing pixel loss to individual samples (e.g., proportional to weight, uniform). [S1]

### Hard Minibatch Size Determination
Techniques for selecting the number of hard samples (fixed ratio vs. adaptive). [S1]

### Pose Error Mitigation
Strategies to handle outliers caused by incorrect camera poses. [S1]

## Operating characteristics
*   **Time Complexity:** Reduces iteration time by focusing updates on a smaller subset of samples. [S1]
*   **Memory Complexity:** Reduces memory usage during training by only storing gradients for the hard minibatch. [S1]
*   **Measured Performance:** On Mip-NeRF-360 scenes, HSM achieves ~1 dB PSNR gain per same training time and converges ~2x faster to a given PSNR level compared to baseline. [S1] Dynamic memory footprint reduced from 300 MB to 160 MB for normal iterations on Nerfacto. [S1]
*   **Strengths:** Faster convergence, lower memory usage, improved quality per unit time. [S1]
*   **Limitations:** Less effective in uniformly challenging scenes; sensitive to pose errors. [S1]
*   **When to Use:** When training time and memory are constraints, and the scene has distinct hard/soft regions. [S1]
*   **When Not to Use:** In scenes with uniform difficulty or significant pose errors. [S1]

## Background
Hard Sample Mining was introduced by J. Korhonen et al. in 2024 in the paper "Efficient NeRF Optimization -- Not All Samples Remain Equally Hard". It was applied to PyTorch reimplementations of Instant-NGP and Nerfacto on Mip-NeRF-360 and NeRF synthetic datasets. [S1]

## Sources
[S1] Efficient NeRF Optimization -- Not All Samples Remain Equally Hard, https://arxiv.org/abs/2408.03193v1, fetched 2026-10-27
