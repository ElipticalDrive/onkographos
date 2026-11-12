# Temporal Sensitivity Pruning


## Overview
Temporal Sensitivity Pruning (TSP) is a method for removing low-impact 3D Gaussians by calculating a temporal sensitivity score that approximates the second-order sensitivity to the $L_2$ loss across all training views and timesteps [S1, unofficial]. It sets itself apart from static pruning methods by aggregating gradients over time, thereby capturing the cumulative contribution of each Gaussian to dynamic scene reconstruction and stabilizing pruning in deformable settings [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The set of 3D Gaussians $\mathcal{G}$, the set of training poses and timesteps $\mathcal{P}_{gt}$, and the deformation field $\mathcal{D}$ (if applicable).
*   **Output:** A binary mask or updated set $\mathcal{G}'$ containing only Gaussians with sensitivity scores above a pruning threshold.

**Data Structures:**
*   **Gaussian State:** Each Gaussian $\mathcal{G}_i$ maintains its parameters (mean $\mu$, covariance $\Sigma$, opacity $\alpha$, spherical harmonics coefficients).
*   **Sensitivity Buffer:** A scalar value $\tilde{U}_{\mathcal{G}_i}$ stored per Gaussian to accumulate the squared gradients.

**Algorithm Steps:**
1.  **Gradient Calculation:** For each Gaussian $\mathcal{G}_i$ and for every training pose $\phi$ and timestep $t \in \mathcal{P}_{gt}$, calculate the gradient of the rendered image $I_{\mathcal{G}_t}(\phi)$ with respect to the projected Gaussian value $g_i$ [S1, unofficial].
    *   Note: When calculating sensitivity, do not clamp negative direction-dependent colors to zero, as this sets partial derivatives to zero for SH coefficients that may still contribute to training images [S2].
2.  **Sensitivity Aggregation:** Square the calculated gradients and sum them over all poses and timesteps to compute the cumulative sensitivity score $\tilde{U}_{\mathcal{G}_i}$ [S1, unofficial]:
    $$
    \tilde{U}_{\mathcal{G}_i} \approx \nabla_{g_i}^2 L_2 \approx \sum_{\phi, t \in \mathcal{P}_{gt}} \left( \nabla_{g_i} I_{\mathcal{G}_t}(\phi) \right)^2
    $$
    where $g_i$ is the value of $\mathcal{G}_i$ projected onto image space [S1, unofficial].
3.  **Pruning:** Periodically (e.g., every $N$ iterations) compare $\tilde{U}_{\mathcal{G}_i}$ against a threshold. Gaussians with low sensitivity scores are removed to reduce neural inference load and boost rendering speed [S1, unofficial].
    *   Gaussians exhibiting zero sensitivity in color parameters are eliminated as they have no impact on training images [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_temporal_sensitivity(gaussians, training_data, deformation_field):
    """
    Computes the TSP score for each Gaussian.
    
    Args:
        gaussians: List of Gaussian objects with parameters (mu, cov, opacity, SH).
        training_data: List of tuples (pose_phi, timestep_t, ground_truth_image).
        deformation_field: Function D(mu, r, s, t) returning deformed parameters.
        
    Returns:
        sensitivity_scores: Dictionary mapping Gaussian ID to scalar TSP score.
    """
    sensitivity_scores = {g.id: 0.0 for g in gaussians}
    
    for g in gaussians:
        total_grad_sq = 0.0
        
        for phi, t, gt_img in training_data:
            # 1. Deform Gaussian for current timestep
            # Note: Deformation updates vary across timesteps, making gradients time-dependent [S1, unofficial]
            deformed_g = deformation_field(g.mu, g.rot, g.scale, t)
            
            # 2. Render image contribution of this Gaussian
            # I_G_t(phi) is the rendered image at pose phi and timestep t [S1, unofficial]
            # We need the gradient of the rendered image w.r.t the projected value g_i
            # In practice, this is computed via backpropagation through the rasterizer
            # Crucial: Do NOT clamp negative colors to zero during this backward pass [S2]
            
            # Forward pass to get rendered color contribution
            rendered_color = render_gaussian(deformed_g, phi)
            
            # Compute L2 loss gradient w.r.t the Gaussian's projected value
            # dL/dg_i = 2 * (rendered_color - gt_pixel)
            # We approximate the second-order sensitivity as the square of this gradient
            # Note: The formula sums over all pixels affected by the Gaussian
            pixel_error = rendered_color - gt_pixel_at_location(deformed_g, phi)
            grad = 2.0 * pixel_error
            
            # Accumulate squared gradient
            # Sum over all pixels covered by the Gaussian's 2D projection
            for pixel in pixels_covered_by(deformed_g, phi):
                total_grad_sq += dot(grad[pixel], grad[pixel])
                
        sensitivity_scores[g.id] = total_grad_sq
        
    return sensitivity_scores

def prune_gaussians(gaussians, sensitivity_scores, threshold):
    """
    Removes Gaussians with sensitivity scores below the threshold.
    """
    active_gaussians = []
    for g in gaussians:
        score = sensitivity_scores[g.id]
        # Prune if score is low (or zero) [S1, unofficial][S2]
        if score > threshold:
            active_gaussians.append(g)
        else:
            # Optional: Log removed Gaussians for debugging
            pass
    return active_gaussians
```

**Parameters:**
*   $\mathcal{P}_{gt}$: The set of training poses and timesteps used for sensitivity aggregation [S1, unofficial].
*   Threshold: The minimum sensitivity score required to retain a Gaussian. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Color Clamping:** Kerbl et al. clamp negative direction-dependent colors to zero, setting partial derivatives to zero in the backward pass. This results in zero sensitivity for SH coefficients that may still contribute to training images. To avoid this, do not clamp colors when calculating sensitivity [S2].
*   **Detail Loss:** Higher pruning thresholds can remove fine details (e.g., small leaves) from the reconstruction [S2].
*   **Floater Detection:** TSP relies on gradients from discrete training frames. It may overlook floaters—unstable Gaussians that appear static in observed views but drift at unseen timestamps. This is addressed by Temporal Sensitivity Sampling (TSS), not TSP itself [S1, unofficial].

## Subtopics

### Sensitivity Calculation and Pruning
A variant method that eliminates Gaussians with no impact on training images by calculating sensitivity using the gradient of total image energy with respect to parameters. It specifically addresses the issue of zero sensitivity caused by color clamping by avoiding the clamping of negative direction-dependent colors during sensitivity calculation. It eliminates splats exhibiting zero sensitivity in color parameters. However, higher pruning thresholds in this variant can lead to the removal of fine details like small leaves [S2].

## Operating characteristics
*   **Time Complexity:** The sensitivity calculation requires iterating over all Gaussians and all training views/timesteps, making it $O(|\mathcal{G}| \cdot |\mathcal{P}_{gt}|)$. This is performed periodically, not at every step.
*   **Memory Complexity:** Requires storing a sensitivity score per Gaussian.
*   **Performance:** Integrating TSP into DeformableGS accelerates rendering by 6.78× on average while maintaining neural-field fidelity and using 10× fewer primitives [S1, unofficial].
*   **Strengths:** Reduces redundant primitives, stabilizes temporal pruning, and lowers deformation cost [S1, unofficial].
*   **Limitations:** May remove fine details if thresholds are too high [S2]. Relies on discrete training frames, potentially missing floaters at unseen timestamps [S1, unofficial].

## Background
TSP was introduced in the context of Speedy Deformable 3D Gaussian Splatting (SpeeDe3DGS) to bridge the efficiency–fidelity gap in dynamic scene reconstruction [S1, unofficial]. It builds on DeformableGS and is designed to remove low-impact Gaussians via temporally aggregated sensitivity analysis [S1, unofficial].

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
[S2] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
