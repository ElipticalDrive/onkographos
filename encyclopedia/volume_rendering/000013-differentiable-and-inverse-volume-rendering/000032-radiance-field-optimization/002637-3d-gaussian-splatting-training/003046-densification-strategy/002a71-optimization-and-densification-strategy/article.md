# Optimization and Densification Strategy

Also called: Modified 3DGS Optimization

## Overview
This method defines the specific optimization loop and adaptive density control procedures for 3D Gaussian Splatting (3DGS) training, particularly when employing non-standard rendering pipelines (e.g., ray-splat intersection or inversion-free splatting) that invalidate the standard screen-space gradient metrics. It solves the problem of aliasing and incorrect densification decisions caused by the absence of standard EWA splatting gradients by substituting alternative metrics (view-space mean gradients, opacity decay) and adapting splitting heuristics. It differs from standard 3DGS [Parent](6234-densification-strategy.md) by replacing opacity resets with exponential decay and using view-space gradients instead of screen-space gradients, and differs from [Training Objective and Densification](7448-training-objective-and-densification.md) by focusing on the implementation details of the densification triggers and metric substitutions.

## How it works
The method integrates optimization steps with periodic densification. It assumes a base 3DGS setup but modifies the densification logic to support hybrid transparency or ray-splat intersection rendering.

**Inputs:**
*   Set of 3D Gaussians $\mathcal{G} = \{\mathcal{G}_i\}$, where each $\mathcal{G}_i$ has mean $\mu_i$, covariance $\Sigma_i$ (or scale/rotation), opacity $\alpha_i$, and color coefficients.
*   Camera poses and intrinsics.
*   Ground truth images $I^{(c)}$.
*   Current training step $t$.

**Outputs:**
*   Updated set $\mathcal{G}'$ with modified parameters and potentially new primitives.

**Data Structures:**
*   Gaussian buffers: GPU arrays storing $\mu, \Sigma, \alpha, c$.
*   Capacity: Pre-allocated buffer size, typically $3 \times$ initial count, to accommodate splits without reallocation [S2, unofficial].

**Parameters:**
*   `refineEvery`: Interval for densification (e.g., 100 steps).
*   `warmupLength`: Steps before densification starts.
*   `stopSplitAt`: Step after which splitting stops.
*   $\lambda_o$: Opacity decay multiplier. Default: $0.9995$ [S1].
*   Decay interval: Steps between opacity decay applications. Default: $50$ [S1].
*   Tile size: For rasterization. Default: $8 \times 8$ to mitigate divergence in insertion sort for per-pixel core tracking [S1].
*   $K$: Core size for blending. Default: $16$ [S1].
*   $\tau_K$: Minimum $\alpha$ for core consideration. Default: $0.05$ [S1].

**Steps:**

1.  **Rendering:** Render current view using the modified splat evaluation (e.g., ray-splat intersection). Compute loss $\mathcal{L}$ against ground truth $I^{(c)}$ [S1].
2.  **Optimization Step:** Compute gradients $\nabla_{\theta} \mathcal{L}$ for all parameters $\theta$ (means, scales, rotations, opacities, colors). Update parameters using an optimizer (e.g., Adam) [S1].
3.  **Densification Trigger:** If $t \ge \text{warmupLength}$ and $t \pmod{\text{refineEvery}} == 0$ and $t < \text{stopSplitAt}$:
    a. **Metric Computation:**
        *   Compute gradient norm for each Gaussian $\mathcal{G}_i$. Since screen-space gradients are unavailable in the modified rendering pipeline, use the gradient of the splat means scaled by half the distance to the camera in view space:
            $$ \text{Score}_i = \left\| \nabla_{\mu_i} \mathcal{L} \right\| \cdot \frac{d_i}{2} $$
            where $d_i$ is the distance from $\mu_i$ to the camera center in view space [S1].
        *   Alternatively, use opacity or visibility scores for pruning [S1].
    b. **Splitting:**
        *   Select Gaussians with high $\text{Score}_i$.
        *   Replace each selected $\mathcal{G}_i$ with two new Gaussians $\mathcal{G}_{i1}, \mathcal{G}_{i2}$.
        *   New means: $\mu_{i1} = \mu_i + \delta$, $\mu_{i2} = \mu_i - \delta$, where $\delta$ is a perturbation along principal axes of $\Sigma_i$ [S1].
        *   New scales: Reduce scale by a factor (e.g., $1.6$) [S1].
        *   Copy color and other attributes.
    c. **Cloning:**
        *   Select Gaussians with high error but low gradient norm (or specific criteria) to fill gaps.
        *   Clone $\mathcal{G}_i$ to $\mathcal{G}_{i'}$ with a small random perturbation to $\mu_{i'}$ [S1].
    d. **Pruning:**
        *   Remove Gaussians with opacity below a threshold or visibility score below a threshold [S1].
    e. **Opacity Management:**
        *   Instead of resetting opacity, apply exponential decay to existing Gaussians every 50 iterations during densification:
            $$ \alpha_i \leftarrow \alpha_i \cdot \lambda_o $$
            where $\lambda_o = 0.9995$ [S1].
        *   Newly split primitives may have initial opacity set to a low value (e.g., logit -1.386) [S1].
4.  **Buffer Management:** Compact buffers to remove pruned Gaussians and ensure capacity [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimization_and_densification_step(gaussians, cameras, gt_images, step, params):
    """
    One step of optimization and potential densification.
    """
    # 1. Render and Compute Loss
    rendered_images = render(gaussians, cameras) # Uses modified splat evaluation
    loss = compute_loss(rendered_images, gt_images)
    
    # 2. Gradient Update
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
    
    # 3. Densification Check
    if (step >= params.warmup_length and 
        step % params.refine_every == 0 and 
        step < params.stop_split_at):
        
        # Compute Metrics
        # Note: Screen-space gradients not available; use view-space mean gradients
        mean_grads = get_gradient_of_means(gaussians, loss) # Shape: (N, 3)
        distances = get_distance_to_camera(gaussians, cameras) # Shape: (N,)
        scores = mean_grads.norm(dim=1) * (distances / 2.0)
        
        # Splitting
        split_candidates = get_top_k_indices(scores, k=params.num_splits)
        new_gaussians = []
        for idx in split_candidates:
            g = gaussians[idx]
            # Split into two
            g1 = clone_gaussian(g)
            g2 = clone_gaussian(g)
            # Perturb means
            g1.mean = g.mean + perturb(g.covariance)
            g2.mean = g.mean - perturb(g.covariance)
            # Reduce scales
            g1.scale = g.scale / 1.6
            g2.scale = g.scale / 1.6
            # Reset opacity for new splits
            g1.opacity = logit_to_opacity(-1.386)
            g2.opacity = logit_to_opacity(-1.386)
            new_gaussians.extend([g1, g2])
        
        # Cloning
        clone_candidates = get_clone_candidates(gaussians) # Criteria not fully specified in [S1], assume high error, low grad
        for idx in clone_candidates:
            g = gaussians[idx]
            g_clone = clone_gaussian(g)
            g_clone.mean = g.mean + random_perturbation()
            new_gaussians.append(g_clone)
        
        # Pruning
        prune_mask = gaussians.opacity < params.opacity_threshold
        gaussians = gaussians[~prune_mask]
        
        # Add new gaussians
        gaussians = concat(gaussians, new_gaussians)
        
        # Opacity Decay
        if step % 50 == 0:
            gaussians.opacity = gaussians.opacity * params.lambda_o
            
        # Buffer Compaction
        gaussians = compact_buffers(gaussians)
        
    return gaussians
```

## Subtopics

### Anisotropy Regularization
A variant that removes structural splitting and training calls, keeping standard densification and pruning. It introduces a log-ratio soft regularization to control anisotropy. Parameters: `lambda_aniso` = 0.001, `aniso_ratio_max` = 5 [S2, unofficial].

### GaussianDreamer
A method combining 2D and 3D diffusion model priors. It uses Shap-E to generate an initial sparse point cloud, optimizes 3DGS using 2D SDS, and applies noisy point growth and color perturbation to densify the sparse cloud. Pitfall: Initial cloud is sparse, and Janus problem may persist if views are sampled one by one [S3].

### Distortion-based Gaussian Splitting Prioritization
A criterion for prioritizing splits based on rate-distortion optimization to maximize PSNR gain. It associates a reconstruction Squared Error $SE_{c,i}$ to each Gaussian:
$$ SE_{c,i} = \sum_{p \in I_c} w_i^{(c)} (R^{(c)}(p) - I^{(c)}(p))^2 $$
Gaussians inducing the highest PSNR gain upon splitting are prioritized [S4].

## Operating characteristics
*   **Time Complexity:** Densification occurs every `refineEvery` steps, not every step, reducing overhead. Splitting and cloning are $O(N)$ where $N$ is the number of Gaussians. Buffer compaction is $O(N)$.
*   **Memory Complexity:** Pre-allocated buffer size is $3 \times$ initial count. Compaction reduces memory usage over time by removing pruned Gaussians [S2, unofficial].
*   **Performance:** Modified approaches (e.g., using ray-splat intersection) can achieve faster training (2.1×) and rendering (1.6×) compared to standard 3DGS [S1].
*   **Strengths:** Adapts 3DGS to non-standard rendering pipelines; prevents over-densification via opacity decay; reduces aliasing with 3D filters [S1].
*   **Limitations:** Requires careful tuning of decay parameters; view-space gradients may be less accurate than screen-space gradients for densification [S1].

## Background
Originated from modifications to the original 3DGS paper [KKLD23] to support hybrid transparency and ray-splat intersection rendering [S1]. The opacity decay strategy was adapted from StopThePop [RSP*24]. The view-space gradient metric was proposed by Moenne-Loccoz et al. [MLMP*24]. The method is used in efficient perspective-correct 3D Gaussian Splatting implementations [S1].

## Sources
[S1] Efficient Perspective‐Correct 3D Gaussian Splatting Using Hybrid Transparency, https://doi.org/10.1111/cgf.70014, fetched 2026-10-30
[S2] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S4] BOGausS: Better Optimized Gaussian Splatting, https://doi.org/10.23919/eusipco63237.2025.11226626, fetched 2026-10-26
