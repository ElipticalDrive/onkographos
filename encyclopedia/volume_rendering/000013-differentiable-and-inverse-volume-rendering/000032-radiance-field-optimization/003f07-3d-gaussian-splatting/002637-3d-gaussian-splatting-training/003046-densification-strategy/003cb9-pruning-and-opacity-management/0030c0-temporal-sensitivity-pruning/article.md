# Temporal Sensitivity Pruning


## Overview
Temporal Sensitivity Pruning (TSP) is a pruning method that removes low-impact Gaussians by calculating a temporal sensitivity score approximating the second-order sensitivity to the L2 loss across all training views [S1, unofficial]. It sets itself apart from static pruning methods by aggregating gradient information over time, thereby identifying Gaussians that are redundant not just in a single view, but across the entire dynamic sequence [S1, unofficial]. This makes it particularly effective for dynamic scene reconstruction where Gaussians may drift or become unstable at unseen timestamps [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The set of active Gaussians $\mathcal{G}_i$, the set of training poses and timesteps $\mathcal{P}_{gt}$, and the deformation field parameters.
*   **Output:** A binary mask or list of Gaussians to be pruned based on their computed sensitivity scores.

**Data Structures:**
*   **Sensitivity Buffer:** A 1D array or buffer storing the scalar TSP score $\tilde{U}_{\mathcal{G}_i}$ for each Gaussian $i$. Size: $N$, where $N$ is the number of active Gaussians.
*   **Training Set:** $\mathcal{P}_{gt}$, a collection of tuples $(\phi, t)$ representing camera poses and timestamps.

**Numbered Steps:**
1.  **Initialize Scores:** For each Gaussian $\mathcal{G}_i$, initialize the cumulative sensitivity score $\tilde{U}_{\mathcal{G}_i} = 0$.
2.  **Iterate Training Views:** For each training pose $\phi$ and timestep $t$ in the set $\mathcal{P}_{gt}$:
    a.  **Forward Pass:** Render the image $I_{\mathcal{G}_t}(\phi)$ using the current Gaussian parameters and deformation field.
    b.  **Gradient Calculation:** Compute the gradient of the rendered image pixel values with respect to the projected Gaussian value $g_i$ for all pixels covered by $\mathcal{G}_i$. Let $\nabla_{g_i} I_{\mathcal{G}_t}(\phi)$ denote this gradient vector.
    c.  **Accumulate:** Square the gradient components and add them to the cumulative score:
        $$ \tilde{U}_{\mathcal{G}_i} \leftarrow \tilde{U}_{\mathcal{G}_i} + \sum_{\text{pixels}} \left( \nabla_{g_i} I_{\mathcal{G}_t}(\phi) \right)^2 $$
3.  **Prune:** Compare $\tilde{U}_{\mathcal{G}_i}$ against a predefined threshold. Gaussians with scores below the threshold are removed from the active set.

**Formula:**
The TSP score $\tilde{U}_{\mathcal{G}_i}$ approximates the second-order sensitivity to the $L_2$ loss:
$$ \tilde{U}_{\mathcal{G}_i} \approx \nabla_{g_i}^2 L_2 \approx \sum_{\phi, t \in \mathcal{P}_{gt}} \left( \nabla_{g_i} I_{\mathcal{G}_t}(\phi) \right)^2 $$
Where:
*   $\tilde{U}_{\mathcal{G}_i}$ is the TSP score for Gaussian $\mathcal{G}_i$.
*   $g_i$ is the value of $\mathcal{G}_i$ projected onto image space.
*   $\mathcal{P}_{gt}$ is the set of training poses and timesteps.
*   $I_{\mathcal{G}_t}(\phi)$ is the rendered image at pose $\phi$ and timestep $t$.

**Parameters:**
*   $\mathcal{P}_{gt}$: The set of training poses and timesteps used for sensitivity aggregation. In dynamic scenes, this includes all available frames [S1, unofficial].
*   Pruning Threshold: Not explicitly defined in the sources. [G] Typically determined empirically; higher thresholds remove more Gaussians but risk losing fine details [S2].

**Edge Cases and Pitfalls:**
*   **Zero Sensitivity due to Clamping:** If negative direction-dependent colors (resulting from SH coefficient evaluation) are clamped to zero during the forward pass, the partial derivatives in the backward pass become zero. This results in a sensitivity of zero for those SH coefficients, even if they contribute to the training images [S2]. To avoid this, do not clamp colors when calculating sensitivity [S2].
*   **Loss of Fine Details:** Higher pruning thresholds can remove fine details (e.g., small leaves) from the reconstruction [S2].
*   **Static Floaters:** TSP relies on gradients from discrete training frames. It may overlook "floaters" — unstable Gaussians that appear static in observed views but drift at unseen timestamps [S1, unofficial]. (Note: This is addressed by the related Temporal Sensitivity Sampling method, not TSP itself).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_temporal_sensitivity_scores(gaussians, training_data, deformation_field):
    """
    Computes the TSP score for each Gaussian.
    
    Args:
        gaussians: List of Gaussian objects, each having a unique ID and parameters.
        training_data: List of tuples (pose, timestep) representing P_gt.
        deformation_field: Function or object to deform Gaussian parameters based on timestep.
        
    Returns:
        scores: Dictionary mapping Gaussian ID to its TSP score.
    """
    scores = {g.id: 0.0 for g in gaussians}
    
    for pose, timestep in training_data:
        # Deform Gaussians for the current timestep
        deformed_gaussians = [g.deform(timestep, deformation_field) for g in gaussians]
        
        # Render image for this pose/timestep
        # Note: In a real implementation, this would be a differentiable rasterization call.
        # We assume render_image returns a tensor and we can backpropagate.
        image = render_image(deformed_gaussians, pose)
        
        # For each Gaussian, compute gradient of image w.r.t projected value g_i
        # This requires tracking which pixels are covered by each Gaussian.
        # Assuming a mask or accumulation buffer is available from the forward pass.
        
        # Compute gradients for all Gaussians simultaneously if possible, or iterate.
        # grad_image_wrt_g is a tensor of shape (N_gaussians, H, W) or similar.
        # However, the formula sums over pixels.
        
        # Efficient implementation:
        # 1. Get gradients of loss w.r.t image pixels (usually just image - target if L2).
        # 2. Backpropagate through rasterizer to get gradients w.r.t Gaussian parameters.
        # 3. Specifically extract gradient w.r.t the projected value g_i.
        
        # Simplified conceptual step:
        # dL_dI = image - target_image (for L2 loss)
        # dI_dg_i = gradient of image w.r.t Gaussian i's projected value
        
        # In practice, we accumulate the squared gradient of the rendered image 
        # w.r.t the Gaussian's contribution.
        
        # Let's assume we have a function get_gradient_wrt_g(gaussian, image_grads)
        # which returns the sum of squared gradients for that Gaussian over all pixels.
        
        for g in deformed_gaussians:
            # Compute gradient of the rendered image w.r.t the projected value g_i
            # This is the core of the sensitivity metric.
            # Note: Do NOT clamp negative colors to zero here to preserve sensitivity [S2].
            grad_g_i = compute_gradient_wrt_projected_value(g, image)
            
            # Square and sum over pixels (already summed in compute_gradient_wrt_projected_value)
            scores[g.id] += torch.sum(grad_g_i ** 2)
            
    return scores

def prune_gaussians(gaussians, scores, threshold):
    """
    Removes Gaussians with TSP scores below the threshold.
    """
    to_prune = [g.id for g in gaussians if scores[g.id] < threshold]
    # Implementation of buffer compaction is handled by the parent method [Parent]
    return [g for g in gaussians if g.id not in to_prune]
```

## Subtopics

### Sensitivity Calculation and Pruning
A variant approach that eliminates Gaussians with no impact on training images by calculating sensitivity using the gradient of total image energy with respect to parameters [S2]. It specifically addresses the issue of color clamping: Kerbl et al. clamp negative direction-dependent colors to zero, setting partial derivatives to zero in the backward pass, which results in zero sensitivity for SH coefficients that may still contribute [S2]. This variant does not clamp colors when calculating sensitivity to avoid this [S2]. It observes that up to 15% of Gaussians exhibit zero sensitivity in color parameters and eliminates them [S2]. A pitfall is that higher pruning thresholds can remove fine details, such as small leaves, from the reconstruction [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot |\mathcal{P}_{gt}| \cdot P)$, where $N$ is the number of Gaussians, $|\mathcal{P}_{gt}|$ is the number of training views/timesteps, and $P$ is the number of pixels covered by each Gaussian. This is significantly more expensive than static pruning as it requires multiple forward/backward passes over the entire training set.
*   **Memory Complexity:** $O(N)$ to store the sensitivity scores. The rendering process requires standard memory for the image buffers.
*   **Measured Performance:** Integrating TSP into DeformableGS accelerates rendering by 6.78× on average while maintaining neural-field fidelity and using 10× fewer primitives [S1, unofficial]. Adding GroupFlow culminates in 13.71× faster rendering and 2.53× shorter training [S1, unofficial].
*   **Strengths:** Effectively reduces redundant primitives in dynamic scenes by considering temporal consistency. Stabilizes temporal pruning and lowers deformation cost [S1, unofficial].
*   **Limitations:** Computationally expensive during the pruning decision phase due to the need to aggregate gradients over all training views. May remove fine details if the threshold is set too high [S2]. Does not inherently address floaters that are stable in training views but unstable in between [S1, unofficial].

## Background
Temporal Sensitivity Pruning was introduced in the context of Speedy Deformable 3D Gaussian Splatting (SpeeDe3DGS) [S1, unofficial]. It builds on DeformableGS and is designed to bridge the efficiency-fidelity gap in dynamic scene reconstruction [S1, unofficial]. The method was developed to address the computational expense of neural inference in deformable Gaussian splatting models [S1, unofficial].

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
[S2] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
