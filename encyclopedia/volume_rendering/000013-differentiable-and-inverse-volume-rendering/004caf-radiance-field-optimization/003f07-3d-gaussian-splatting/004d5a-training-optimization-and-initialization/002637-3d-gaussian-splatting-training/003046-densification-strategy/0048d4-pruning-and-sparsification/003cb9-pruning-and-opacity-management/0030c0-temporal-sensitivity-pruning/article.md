# Temporal Sensitivity Pruning


## Overview
Temporal Sensitivity Pruning (TSP) is a pruning method that removes low-impact 3D Gaussians by calculating a temporal sensitivity score, which approximates the second-order sensitivity to the $L_2$ loss across all training views and timesteps [S1, unofficial]. Unlike static pruning methods that rely solely on current opacity or gradient magnitude, TSP aggregates gradients over time to identify Gaussians that are redundant for reconstructing the dynamic scene [S1, unofficial]. This approach is particularly effective in deformable or dynamic Gaussian splatting, where it reduces neural inference load and rendering latency by eliminating primitives that do not contribute significantly to the temporal reconstruction [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G} = \{\mathcal{G}_1, \dots, \mathcal{G}_N\}$, where each $\mathcal{G}_i$ has parameters including mean, covariance, opacity, and spherical harmonic (SH) coefficients. A set of training poses and timesteps $\mathcal{P}_{gt}$ [S1, unofficial].
*   **Output:** A binary mask or list of Gaussians to be pruned based on their computed sensitivity scores $\tilde{U}_{\mathcal{G}_i}$ [S1, unofficial].

**Data Structures:**
*   **Sensitivity Buffer:** An array of size $N$ storing the scalar score $\tilde{U}_{\mathcal{G}_i}$ for each Gaussian.
*   **Training Set:** A list of tuples $(\phi, t)$ representing camera poses and timestamps [S1, unofficial].

**Algorithm Steps:**
1.  **Gradient Calculation:** For each Gaussian $\mathcal{G}_i$ and for every training pose $\phi$ and timestep $t \in \mathcal{P}_{gt}$, calculate the gradient of the rendered image $I_{\mathcal{G}_t}(\phi)$ with respect to the projected value $g_i$ of the Gaussian [S1, unofficial].
    *   Note: $g_i$ represents the contribution of $\mathcal{G}_i$ to the image space at the specific pose and time [S1, unofficial].
2.  **Sensitivity Aggregation:** Square the gradients obtained in Step 1 and sum them over all poses and timesteps in $\mathcal{P}_{gt}$ to compute the cumulative sensitivity score $\tilde{U}_{\mathcal{G}_i}$ [S1, unofficial].
3.  **Pruning Decision:** Compare $\tilde{U}_{\mathcal{G}_i}$ against a threshold (or select the lowest scoring Gaussians). Periodically prune Gaussians with low sensitivity scores during training [S1, unofficial].

**Formula:**
The TSP score $\tilde{U}_{\mathcal{G}_i}$ for a Gaussian $\mathcal{G}_i$ is defined as [S1, unofficial]:

$$
\tilde{U}_{\mathcal{G}_i} \approx \nabla_{g_i}^2 L_2 \approx \sum_{\phi, t \in \mathcal{P}_{gt}} \left( \nabla_{g_i} I_{\mathcal{G}_t}(\phi) \right)^2
$$

Where:
*   $\tilde{U}_{\mathcal{G}_i}$ is the temporal sensitivity score for Gaussian $\mathcal{G}_i$.
*   $\nabla_{g_i}^2 L_2$ is the approximation of the second-order sensitivity to the $L_2$ loss.
*   $g_i$ is the value of $\mathcal{G}_i$ projected onto image space.
*   $\mathcal{P}_{gt}$ is the set of training poses and timesteps.
*   $I_{\mathcal{G}_t}(\phi)$ is the rendered image at pose $\phi$ and timestep $t$.

**Parameters:**
*   $\mathcal{P}_{gt}$: The set of training poses and timesteps used for sensitivity aggregation. The size of this set determines the computational cost of the pruning step [S1, unofficial].
*   Pruning Threshold: Not explicitly defined in the sources as a fixed number. The method states that low-contributing Gaussians are pruned, implying a threshold or rank-based selection is used [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_temporal_sensitivity(Gaussians, training_data):
    """
    Computes the Temporal Sensitivity Pruning score for each Gaussian.
    
    Args:
        Gaussians: List of Gaussian objects, each with a method to project to image space.
        training_data: List of tuples (pose, timestep).
        
    Returns:
        sensitivity_scores: List of float scores, one per Gaussian.
    """
    n_gaussians = len(Gaussians)
    sensitivity_scores = [0.0] * n_gaussians
    
    # Iterate over all training views and timesteps
    for pose, timestep in training_data:
        # Render the scene or compute gradients for this specific view/time
        # Note: In practice, this requires backpropagation through the rasterizer
        # to get d(Image) / d(g_i) for each Gaussian i.
        
        # Placeholder for the gradient computation step.
        # The source [S1, unofficial] defines the gradient as nabla_{g_i} I_{G_t}(phi).
        # This implies we need the derivative of the rendered pixel values 
        # with respect to the Gaussian's projected contribution.
        
        gradients = compute_gradients_for_view(Gaussians, pose, timestep)
        
        # Accumulate squared gradients
        for i, grad in enumerate(gradients):
            sensitivity_scores[i] += grad ** 2
            
    return sensitivity_scores

def apply_temporal_pruning(Gaussians, sensitivity_scores, threshold=None):
    """
    Prunes Gaussians with sensitivity scores below the threshold.
    
    Args:
        Gaussians: List of Gaussian objects.
        sensitivity_scores: List of scores from compute_temporal_sensitivity.
        threshold: Float. If None, a fixed percentage or absolute value logic 
                   (not specified in sources) is assumed.
                   
    Returns:
        remaining_gaussians: List of Gaussian objects that are not pruned.
    """
    remaining = []
    # Implementation detail: The exact thresholding logic (fixed value vs percentile)
    # is not given in the sources [S1, unofficial]. We assume a standard threshold check.
    
    for i, gaussian in enumerate(Gaussians):
        score = sensitivity_scores[i]
        # [G] Guard: Ensure score is valid
        if score > threshold:
            remaining.append(gaussian)
        # Else: Gaussian is pruned
    return remaining
```

**Edge Cases and Pitfalls:**
*   **Dynamic Deformation:** Since deformation updates vary across timesteps, the gradients are inherently time-dependent. This captures second-order effects of deformations but requires consistent deformation field evaluation [S1, unofficial].
*   **Computational Cost:** Calculating gradients for all training poses and timesteps for every Gaussian can be expensive. The method is applied periodically to mitigate this [S1, unofficial].

## Subtopics

### Sensitivity Calculation and Pruning
A variant method that eliminates Gaussians with no impact on training images by calculating sensitivity using the gradient of total image energy with respect to parameters [S2]. Unlike the standard TSP which focuses on $L_2$ loss gradients, this approach emphasizes the total image energy [S2]. A critical implementation detail is that colors should **not** be clamped to zero when calculating sensitivity to avoid zero sensitivity for SH coefficients that may still contribute to training images [S2]. Gaussians exhibiting zero sensitivity in color parameters are eliminated [S2]. A known pitfall is that higher pruning thresholds can remove fine details (e.g., small leaves) from the reconstruction [S2].

## Operating characteristics
*   **Time Complexity:** The pruning step requires computing gradients for all Gaussians across all training poses and timesteps in $\mathcal{P}_{gt}$. This is $O(N \cdot |\mathcal{P}_{gt}|)$ gradient computations per pruning iteration [S1, unofficial].
*   **Memory Complexity:** Requires storing the sensitivity scores for all $N$ Gaussians, i.e., $O(N)$ [S1, unofficial].
*   **Performance:** Integrating TSP into DeformableGS accelerates rendering by 6.78× on average while maintaining neural-field fidelity and using 10× fewer primitives [S1, unofficial].
*   **Strengths:** Reduces neural inference load and boosts rendering speed by removing redundant primitives [S1, unofficial]. Stabilizes temporal pruning in dynamic scenes [S1, unofficial].
*   **Limitations:** May remove fine details if the pruning threshold is set too high [S2]. Relies on gradients from discrete training frames, which may overlook floaters (addressed by Temporal Sensitivity Sampling, a sibling method) [S1, unofficial].

## Background
Temporal Sensitivity Pruning was introduced in the context of **SpeeDe3DGS** (Speedy Deformable 3D Gaussian Splatting) [S1, unofficial]. It builds on **DeformableGS** and is designed to bridge the efficiency-fidelity gap in dynamic 3D Gaussian Splatting [S1, unofficial]. The method was evaluated on the **MonoDyGauBench** dataset, consisting of 50 dynamic scenes [S1, unofficial]. The work aims to achieve neural-field fidelity at rendering speeds comparable to non-neural motion models [S1, unofficial].

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
[S2] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
