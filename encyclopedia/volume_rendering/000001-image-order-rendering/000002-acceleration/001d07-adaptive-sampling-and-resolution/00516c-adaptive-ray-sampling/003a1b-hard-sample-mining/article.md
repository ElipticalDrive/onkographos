# Hard Sample Mining

Also called: HSM

## Overview
Hard Sample Mining (HSM) is a training acceleration technique for neural radiance fields (NeRF) and similar implicit representations that reduces iteration time and memory usage by subsampling a "hard minibatch" of point samples [S1]. It employs propagated pixel loss to rank point samples and subsequently subsamples a hard minibatch, estimated to retain most of the update information [S1]. Unlike methods that simply prune low-weight samples (e.g., Nerfacc), HSM uses the magnitude of the reconstruction error to prioritize challenging regions and skip well-reconstructed ones, thereby avoiding redundant gradient computation [S1][S2, unofficial].

## How it works
**Inputs:**
*   A batch of rays or point samples $\mathcal{S}$ associated with training images.
*   Propagated pixel loss values $L_i$ for each sample $i \in \mathcal{S}$, derived from the difference between rendered and ground-truth pixels [S1].
*   Total batch size $B$ [S1].

**Outputs:**
*   A subsampled set of hard samples $\mathcal{S}_{hard} \subset \mathcal{S}$ used for the backward pass and parameter update [S1].

**Data Structures:**
*   **Sample List:** A list of objects or tuples containing the sample index, its associated pixel coordinates, and its computed loss value $L_i$.
*   **Ranking Array:** An array of indices sorted by $L_i$ in descending order.

**Steps:**
1.  **Forward Pass:** Render the current batch of rays using the current network state to obtain predicted colors $\hat{C}$ [S1].
2.  **Loss Computation:** Compute the pixel-wise loss $\ell_i$ for each sample $i$ against the ground truth $C_i$ (e.g., MSE or L1) [S1].
3.  **Ranking:** Sort all samples in the batch $\mathcal{S}$ based on their loss values $\ell_i$ in descending order [S1].
4.  **Subsampling:** Select the top $k$ samples from the sorted list to form the hard minibatch $\mathcal{S}_{hard}$, where $k$ is determined by the target batch size $B$ or a fixed subsampling ratio [S1].
    *   *Note:* In the Instant-NGP baseline, $B=220$ [S1].
5.  **Backward Pass:** Compute gradients and update network parameters using only the samples in $\mathcal{S}_{hard}$ [S1].

**Parameters:**
*   $B$: Total batch size. Typical values: 220 for Instant-NGP, 218 for Nerfacto [S1].
*   Subsampling Ratio: Not explicitly defined as a fixed constant in the sources, but implied by the selection of a "hard minibatch" from the full batch [S1].

**Edge Cases and Pitfalls:**
*   **Uniformly Challenging Content:** Scenes with unbounded backgrounds or multiple outward-facing objects result in a more uniform pixel loss distribution, limiting computational efficiency gains [S1].
*   **Pose Errors:** Severe pose mistakes in the dataset can produce "impossible to learn" point samples that consistently have high loss and crowd the hard minibatch, potentially degrading training stability [S1]. Mitigation strategies include uncertainty or pose estimation techniques [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_sample_mining_forward_backward(network, rays, ground_truth, batch_size):
    """
    Performs a forward pass, ranks samples by loss, and performs backward pass
    on the hard subsample.
    
    Args:
        network: The neural network model.
        rays: List of ray origins and directions for the batch.
        ground_truth: Ground truth pixel colors for the rays.
        batch_size: Target number of samples for the hard minibatch.
        
    Returns:
        loss: Scalar loss value of the hard minibatch.
    """
    # 1. Forward Pass
    # Render colors for all rays in the batch
    predicted_colors = network.forward(rays)
    
    # 2. Loss Computation
    # Compute per-sample loss (e.g., Mean Squared Error)
    # losses[i] corresponds to the loss for ray i
    losses = compute_pixel_loss(predicted_colors, ground_truth)
    
    # 3. Ranking
    # Get indices that would sort the losses in descending order
    # This ranks samples from hardest (highest loss) to easiest (lowest loss)
    sorted_indices = argsort_descending(losses)
    
    # 4. Subsampling
    # Select the top 'batch_size' indices to form the hard minibatch
    hard_indices = sorted_indices[:batch_size]
    
    # 5. Backward Pass
    # Gather the predicted colors and ground truth for the hard samples only
    hard_predicted = predicted_colors[hard_indices]
    hard_ground_truth = ground_truth[hard_indices]
    
    # Compute loss on the hard minibatch
    hard_loss = compute_pixel_loss(hard_predicted, hard_ground_truth)
    
    # Perform gradient descent using only the hard samples
    # This reduces memory usage and iteration time
    hard_loss.backward()
    
    return hard_loss
```

## Operating characteristics
*   **Time Complexity:** Reduces iteration time by focusing updates on a subsampled hard minibatch rather than the full batch [S1].
*   **Memory Complexity:** Reduces dynamic memory footprint during training. For example, dynamic memory usage dropped from 300 MB to 160 MB for normal iterations in Nerfacto [S1].
*   **Performance:** On Mip-NeRF-360 scenes, HSM converges ~2x faster to a given PSNR level and achieves ~1 dB gain per same training time compared to the baseline [S1].
*   **Strengths:** Effective for scenes with centrally placed objects against easy backgrounds where loss distribution is skewed [S1].
*   **Limitations:** Less effective in uniformly challenging scenes (e.g., unbounded scenes) where loss distribution is uniform [S1]. Sensitive to severe pose errors in the dataset [S1].
*   **When to use:** When training time and memory are bottlenecks, and the scene has distinct hard/easy regions.
*   **When not to use:** When the dataset has significant pose errors or the scene is uniformly complex.

## Background
Hard Sample Mining was introduced by J. Korhonen et al. in the paper "Efficient NeRF Optimization -- Not All Samples Remain Equally Hard" [S1]. The method was applied to PyTorch reimplimentations of Instant-NGP and Nerfacto on Mip-NeRF-360 and NeRF synthetic datasets [S1]. It differs from Nerfacc's approach of pruning low-weight samples by using propagated pixel loss to rank and subsample a hard minibatch [S1].

## Sources
[S1] Efficient NeRF Optimization -- Not All Samples Remain Equally Hard, https://arxiv.org/abs/2408.03193v1, fetched 2026-10-27
[S2] TurboGS: Accelerating 3D Gaussian Splatting via Error-Guided Sparse Pixel Sampling and Optimization, https://doi.org/10.48550/arXiv.2606.15924, fetched 2026-10-29 (unofficial)
