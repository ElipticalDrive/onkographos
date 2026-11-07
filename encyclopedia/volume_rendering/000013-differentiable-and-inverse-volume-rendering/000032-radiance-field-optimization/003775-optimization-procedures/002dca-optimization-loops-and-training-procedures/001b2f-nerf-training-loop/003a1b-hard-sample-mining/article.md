# Hard Sample Mining

Also called: HSM

## Overview
Hard Sample Mining (HSM) is an optimization procedure for neural radiance fields that improves training efficiency and reconstruction quality by selectively updating the network based on pixel-level error. It solves the problem of uniform computational waste in standard NeRF training, where samples contributing little to the gradient are treated equally to those with high error. Unlike sibling methods such as [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md) or [Nerfacc](https://github.com/NVlabs/nerfacc) which prune low-weight samples based on density or transmittance, HSM ranks samples using the propagated pixel loss to subsample a "hard" minibatch that retains most of the update information [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A batch of rays, each with a set of 3D point samples $\{x_i\}$, their corresponding densities $\sigma_i$ and colors $c_i$, and the rendered pixel color $C$ compared against the ground truth pixel color $C^*$.
*   **Output:** A subsampled set of point samples (the hard minibatch) used for the backward pass and parameter update.

**Data Structures:**
The method operates on the per-sample loss values. For a batch of $N$ rays, let $L_i$ be the pixel loss for ray $i$. The loss is typically the Mean Squared Error (MSE) between the rendered and ground truth colors:
$$ L_i = \| C_i - C^*_i \|^2 $$
The algorithm maintains a list of all point samples across the batch, indexed by $(ray\_idx, sample\_idx)$, associated with their computed loss $L_{i,j}$.

**Steps:**
1.  **Compute Pixel Loss:** For every ray in the current training batch, render the color and compute the pixel-wise loss $L_i$ against the ground truth image.
2.  **Propagate Loss to Samples:** Assign the ray's pixel loss $L_i$ to every point sample $j$ along that ray. Let $l_{i,j} = L_i$. This step assumes that the error at the pixel level is attributable to the samples contributing to that pixel [S1].
3.  **Rank Samples:** Sort all point samples in the batch in descending order of their assigned loss $l_{i,j}$.
4.  **Subsample Hard Minibatch:** Select the top $K$ samples from the sorted list, where $K$ is determined by the target batch size $B$. The specific selection strategy (e.g., uniform top-$K$ or weighted) is not explicitly detailed beyond "subsamples a hard minibatch," but the goal is to retain most update information [S1].
5.  **Compute Gradients:** Perform the backward pass (gradient computation) only for the selected hard minibatch. The gradients for the network parameters are computed based on the loss of these specific samples.
6.  **Update Parameters:** Apply the optimizer step using the computed gradients.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_sample_mining(rays, ground_truth_colors, network, batch_size):
    """
    rays: List of Ray objects, each containing a list of 3D points and associated densities/colors.
    ground_truth_colors: Tensor of shape (num_rays, 3) containing GT pixel colors.
    network: The NeRF network model.
    batch_size: The target number of samples for the hard minibatch update.
    """
    num_rays = len(rays)
    
    # 1. Forward pass and pixel loss computation
    rendered_colors = []
    ray_losses = []
    
    for ray in rays:
        # Render the ray using the current network state
        # This returns the final pixel color C_i
        color_i = network.render(ray)
        rendered_colors.append(color_i)
        
        # Compute pixel loss L_i = || C_i - C*_i ||^2
        gt_color_i = ground_truth_colors[rays.index(ray)]
        loss_i = torch.mean((color_i - gt_color_i) ** 2)
        ray_losses.append(loss_i)
        
    rendered_colors = torch.stack(rendered_colors)
    ray_losses = torch.tensor(ray_losses)
    
    # 2. Propagate loss to individual samples
    # Create a list of all samples with their associated ray loss
    all_samples = []
    for ray_idx, ray in enumerate(rays):
        # ray.points is a list of 3D points for this ray
        # We assume the network has already computed densities/colors for these points
        # or we re-evaluate them. In practice, we access the stored forward pass values.
        for sample_idx, point in enumerate(ray.points):
            # Assign the ray's pixel loss to this sample
            sample_loss = ray_losses[ray_idx]
            all_samples.append({
                'ray_idx': ray_idx,
                'sample_idx': sample_idx,
                'point': point,
                'loss': sample_loss,
                'ray': ray
            })
            
    # 3. Rank samples by loss (descending)
    all_samples.sort(key=lambda x: x['loss'], reverse=True)
    
    # 4. Subsample hard minibatch
    # Select the top 'batch_size' samples
    hard_minibatch = all_samples[:batch_size]
    
    # 5. Compute gradients for the hard minibatch
    # We need to compute the gradient of the loss with respect to network parameters
    # only for these specific samples.
    
    # Re-evaluate or access the forward pass values for the hard samples
    # to compute the specific loss contribution for the backward pass.
    # Note: In a standard autograd framework, we would typically compute the 
    # total loss and then mask the gradient, but HSM explicitly subsamples.
    # Here we simulate the subsampled backward pass.
    
    total_loss = 0
    for sample_info in hard_minibatch:
        ray = sample_info['ray']
        point = sample_info['point']
        
        # Get the rendered color for this specific sample's contribution?
        # Actually, the loss is defined at the pixel level.
        # To update the network, we need the gradient of the pixel loss w.r.t parameters.
        # The pixel loss depends on all samples in the ray.
        # However, HSM updates based on the hard samples.
        # The paper states: "subsamples a hard minibatch... reduces iteration time".
        # This implies we only backpropagate through the hard samples?
        # Or we use the hard samples to weight the update?
        # The text says "retains most of the update information".
        # Standard implementation: Compute loss for the hard samples' rays,
        # but only consider the hard samples in the integration?
        # Let's stick to the description: "subsamples a hard minibatch".
        # This usually means we treat the hard samples as the effective batch.
        
        # Re-render the ray for the hard sample's ray index, but only integrate
        # over the hard samples? Or just use the existing forward pass?
        # Given the constraint of "plain text", we assume the standard 
        # "hard example mining" logic:
        # 1. Compute loss for all.
        # 2. Select top K samples.
        # 3. Backpropagate the loss associated with those samples.
        
        # Since the loss is pixel-wise, we must backpropagate the pixel loss
        # for the rays containing the hard samples.
        pass 
        
    # Simplified PyTorch-like implementation of the update step:
    # We create a tensor of losses for the hard samples' rays.
    # But we must be careful: if multiple hard samples come from the same ray,
    # we don't double count the ray loss.
    
    hard_ray_indices = list(set([s['ray_idx'] for s in hard_minibatch]))
    hard_ray_losses = ray_losses[hard_ray_indices]
    
    # Compute gradients
    hard_ray_losses.sum().backward()
    
    # 6. Update parameters
    optimizer.step()
    optimizer.zero_grad()
```

**Parameters:**
*   $B$: Total batch size for the hard minibatch. Typical values are 220 for Instant-NGP and 218 for Nerfacto [S1].

**Edge Cases and Pitfalls:**
*   **Uniformly Challenging Content:** In scenes with unbounded backgrounds or multiple outward-facing objects, the pixel loss distribution is more uniform. This limits the computational efficiency gains because there are fewer distinct "hard" samples to prioritize [S1].
*   **Pose Errors:** Severe mistakes in camera pose estimation can produce "impossible to learn" point samples. These samples will consistently have high loss and crowd the hard minibatch, potentially destabilizing training [S1]. Mitigation techniques like uncertainty or pose estimation can be applied [S1].

## Operating characteristics
*   **Time Complexity:** Reduces iteration time by focusing gradient updates on a subsampled set of points rather than the full batch. The ranking step adds overhead, but the reduced backward pass size yields net gains [S1].
*   **Memory Complexity:** Reduces dynamic memory footprint. For example, memory usage dropped from 300 MB to 160 MB for normal iterations in Nerfacto [S1].
*   **Performance:** On Mip-NeRF-360 scenes, HSM converges approximately 2x faster to a given PSNR level and achieves a ~1 dB validation PSNR gain per same training time compared to the baseline [S1].
*   **Strengths:** Significant speedup and memory reduction; improved convergence speed.
*   **Limitations:** Less effective in scenes with uniform loss distribution; sensitive to severe pose errors.
*   **When to use:** When training time and memory are constrained, and the scene has distinct hard-to-render regions (e.g., centrally placed objects against easy backgrounds).
*   **When not to use:** In scenes with uniformly challenging content or when camera poses are unreliable.

## Background
Hard Sample Mining was introduced by J. Korhonen et al. in the paper "Efficient NeRF Optimization -- Not All Samples Remain Equally Hard" (2024) [S1]. The method was applied to PyTorch reimplmentations of Instant-NGP and Nerfacto on Mip-NeRF-360 and NeRF synthetic datasets [S1]. It differs from Nerfacc's approach of pruning low-weight samples by using propagated pixel loss to rank and subsample [S1].

## Sources
[S1] Efficient NeRF Optimization -- Not All Samples Remain Equally Hard, https://arxiv.org/abs/2408.03193v1, fetched 2026-10-27
