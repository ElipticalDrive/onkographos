# Hard Sample Mining

Also called: HSM

## Overview
Hard Sample Mining (HSM) is a training optimization technique for neural radiance fields that reduces iteration time and memory usage by subsampling a "hard minibatch" of point samples based on propagated pixel loss [S1]. Unlike methods that prune low-weight samples (e.g., Nerfacc), HSM ranks all samples by their contribution to the training error and retains only the most informative ones, aiming to preserve update information while discarding easy samples [S1]. It builds on [Adaptive Ray Sampling](14469-adaptive-ray-sampling.md) by applying adaptation at the training iteration level rather than the ray-casting level.

## How it works
**Inputs:**
*   A batch of rays, each containing $N$ point samples $\{x_i\}_{i=1}^N$.
*   The rendered color $C$ and accumulated opacity $\alpha$ for each ray.
*   The ground truth color $C_{gt}$ for the corresponding pixel.
*   The total batch size $B$ (number of rays per iteration).

**Outputs:**
*   A subset of rays and their associated samples (the hard minibatch) used to compute the gradient update.

**Data Structures:**
*   **Loss Array:** A 1D array of size $B$ storing the pixel-wise loss $L_j$ for each ray $j \in \{1, \dots, B\}$.
*   **Sample Indices:** A mapping from ray index to the indices of samples within that ray.

**Parameters:**
*   $B$: Total batch size. Typical values: 220 for Instant-NGP, $2^{18}$ for Nerfacto [S1].
*   $k$: The number of hard samples/rays to retain. The sources do not specify the exact formula for $k$, but imply it is a subsample of $B$ [S1].

**Steps:**
1.  **Forward Pass:** Render the batch of rays using the current network state to obtain predicted colors $C$ and accumulated opacities $\alpha$ [S1].
2.  **Loss Computation:** Compute the pixel-wise loss $L_j$ for each ray $j$ in the batch, typically using Mean Squared Error (MSE) between $C_j$ and $C_{gt, j}$ [S1].
3.  **Ranking:** Sort the rays in descending order of their loss $L_j$. The rays with the highest loss are considered "hard" [S1].
4.  **Subsampling:** Select the top $k$ rays (or a subset of samples from these rays) to form the hard minibatch. The exact selection strategy (e.g., selecting all samples from the top $k$ rays vs. selecting a fixed number of hard samples across all rays) is not explicitly detailed, but the goal is to retain most of the update information [S1].
5.  **Backward Pass:** Compute gradients and update network parameters using only the hard minibatch [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_sample_mining(batch_rays, batch_gt_colors, network, batch_size_B):
    """
    Performs one iteration of Hard Sample Mining.
    
    Args:
        batch_rays: List of rays, each containing N samples.
        batch_gt_colors: Ground truth colors for the rays.
        network: The NeRF network model.
        batch_size_B: Total number of rays in the batch.
        
    Returns:
        None (updates network parameters)
    """
    # Step 1: Forward pass
    # Render all rays to get predicted colors and opacities
    # Note: In practice, this might involve proposal networks, but HSM operates on the final samples
    predicted_colors, accumulated_opacities = network.render(batch_rays)
    
    # Step 2: Compute pixel-wise loss for each ray
    # Loss is typically MSE between predicted and ground truth color
    losses = []
    for j in range(batch_size_B):
        # Compute loss for ray j
        # L_j = || C_j - C_gt_j ||^2
        loss_j = torch.mean((predicted_colors[j] - batch_gt_colors[j]) ** 2)
        losses.append(loss_j)
    
    losses = torch.tensor(losses)
    
    # Step 3: Rank rays by loss (descending)
    # Sort indices such that highest loss comes first
    sorted_indices = torch.argsort(losses, descending=True)
    
    # Step 4: Subsample hard minibatch
    # Select top k rays. The exact value of k is not specified in sources.
    # We assume k is a fraction of B or a fixed small number.
    # For implementation, we select the top half of rays as a placeholder for k.
    # [G] A reasonable heuristic for k is B/2 or a fixed number like 64, 
    # but the source does not specify. We will select the top 50% for demonstration.
    k = batch_size_B // 2
    hard_indices = sorted_indices[:k]
    
    # Extract the hard rays and their corresponding ground truth colors
    hard_rays = [batch_rays[i] for i in hard_indices]
    hard_gt_colors = [batch_gt_colors[i] for i in hard_indices]
    
    # Step 5: Backward pass on hard minibatch
    # Render the hard rays again (or reuse forward pass if cached)
    # Note: In a real implementation, you would avoid redundant forward passes
    # by caching the forward pass results for the selected rays.
    hard_predicted_colors, _ = network.render(hard_rays)
    
    # Compute loss on hard minibatch
    hard_losses = []
    for j in range(len(hard_rays)):
        loss_j = torch.mean((hard_predicted_colors[j] - hard_gt_colors[j]) ** 2)
        hard_losses.append(loss_j)
    
    hard_loss = torch.mean(torch.tensor(hard_losses))
    
    # Update network parameters
    hard_loss.backward()
    # Optimizer step would follow here
```

**Edge Cases and Pitfalls:**
*   **Uniformly Challenging Content:** Scenes with unbounded backgrounds or multiple outward-facing objects have uniform pixel loss distributions, limiting the efficiency gains of HSM [S1].
*   **Pose Errors:** Severe pose mistakes in the dataset can create "impossible to learn" samples that consistently rank as hard, crowding the hard minibatch and degrading performance [S1]. Mitigation strategies like uncertainty or pose estimation are suggested but not part of the core HSM algorithm [S1].

## Operating characteristics
*   **Time Complexity:** Reduces iteration time by computing gradients on a smaller subset of samples [S1].
*   **Memory Complexity:** Reduces dynamic memory footprint. For example, from 300 MB to 160 MB for normal iterations in Nerfacto [S1].
*   **Performance:** On Mip-NeRF-360 scenes, HSM converges ~2x faster to a given PSNR level and achieves ~1 dB gain per same training time compared to baseline [S1].
*   **Strengths:** Significant reduction in training time and memory usage; improved convergence speed [S1].
*   **Limitations:** Less effective in scenes with uniform difficulty; sensitive to pose errors [S1].
*   **When to Use:** When training time and memory are constraints, and the scene has varying levels of difficulty (e.g., sharp edges, complex textures) [S1].
*   **When Not to Use:** In scenes with uniformly challenging content or when pose accuracy is poor [S1].

## Background
Hard Sample Mining was introduced by J. Korhonen et al. in the paper "Efficient NeRF Optimization -- Not All Samples Remain Equally Hard" (2024) [S1]. It was applied to PyTorch reimplementations of Instant-NGP and Nerfacto on Mip-NeRF-360 and synthetic NeRF datasets [S1]. The method differs from Nerfacc's pruning approach by using propagated pixel loss to rank and subsample a hard minibatch [S1].

## Sources
[S1] Efficient NeRF Optimization -- Not All Samples Remain Equally Hard, https://arxiv.org/abs/2408.03193v1, fetched 2026-10-27
