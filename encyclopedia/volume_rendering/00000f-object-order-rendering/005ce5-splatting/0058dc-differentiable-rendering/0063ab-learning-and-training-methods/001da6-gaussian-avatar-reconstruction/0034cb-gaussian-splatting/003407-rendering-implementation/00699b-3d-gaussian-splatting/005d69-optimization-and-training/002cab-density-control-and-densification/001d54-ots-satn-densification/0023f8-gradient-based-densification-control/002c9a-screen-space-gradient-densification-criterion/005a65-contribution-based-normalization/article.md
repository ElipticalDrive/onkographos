# Contribution-Based Normalization

Also called: Average Contribution Normalization, contribution-weighted densification, mean contribution normalization

## Overview
Contribution-Based Normalization is a densification criterion for 3D Gaussian Splatting that normalizes the accumulated photometric gradient by the Gaussian's average rendering contribution across views, rather than by the raw number of visible views or pixels [S1, unofficial]. It solves the problem of occlusion-induced bias where standard view-count normalization stalls densification for Gaussians that are frequently occluded but still require refinement in visible regions [S1, unofficial]. Unlike its sibling [Screen-Space Gradient Densification Criterion](5303-screen-space-gradient-densification-criterion.md) which relies on gradient magnitude thresholds, this method scales the gradient signal by the mean blending weight $\bar{w}_i$, ensuring that the densification score reflects the actual influence of the Gaussian on the image rather than its geometric footprint or visibility count [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** For each Gaussian $i$, the accumulated gradient vector $\mathbf{g}_i$ over all training views, and the set of per-pixel contribution weights $w_{i,p}$ for all pixels $p$ where the Gaussian contributes to the final color.
*   **Output:** A scalar densification score $\gamma_i$. If $\gamma_i$ exceeds a threshold $\tau$, the Gaussian is split or cloned.

**Data Structures:**
*   **Accumulated Gradient $\mathbf{g}_i$:** A 2D vector $(g_{i,x}, g_{i,y})$ storing the sum of gradients with respect to the projected screen-space mean $\mu_i$ across all views.
*   **Contribution Weights $w_{i,p}$:** A scalar value per pixel $p$ in the view, defined as $w_{i,p} = \alpha_{i,p} T_{i,p}$, where $\alpha_{i,p}$ is the Gaussian's opacity at pixel $p$ and $T_{i,p}$ is the transmittance reaching the Gaussian [S1, unofficial].

**Formulas:**
1.  **Projected Footprint:** The unnormalized Gaussian kernel at pixel $p$ is:
    $$ G_{i,p} = \exp\left(-\frac{1}{2} (p - \mu_i)^T \Sigma_i^{-1} (p - \mu_i)\right) $$
    where $\mu_i$ is the projected screen-space mean and $\Sigma_i$ is the projected 2D covariance matrix [S1, unofficial].

2.  **Per-Pixel Gradient Component:** The gradient contribution for the x-component (y is analogous) at pixel $p$ in a specific view is:
    $$ g_{i,x,p} = \frac{\partial L_p}{\partial C_p} \cdot (c_i - A_{i,p}) \cdot w_{i,p} \cdot \frac{\partial \ln(G_{i,p})}{\partial \mu_{i,x}} $$
    where:
    *   $\frac{\partial L_p}{\partial C_p}$ is the photometric error gradient at pixel $p$.
    *   $c_i$ is the color of Gaussian $i$.
    *   $A_{i,p}$ is the accumulated color from Gaussians rendered *before* $i$ (background/transmittance accumulation).
    *   $w_{i,p} = \alpha_{i,p} T_{i,p}$ is the contribution weight [S1, unofficial].
    *   $\frac{\partial \ln(G_{i,p})}{\partial \mu_{i,x}}$ is the geometric steering term, derived from the Gaussian's shape.

3.  **Accumulated Gradient:** The total gradient for Gaussian $i$ in a view is the sum over all pixels $p$:
    $$ \mathbf{g}_{i,v} = \sum_p \mathbf{g}_{i,p} $$
    The global accumulated gradient $\mathbf{g}_i$ is the sum of $\mathbf{g}_{i,v}$ over all views $v$ where the Gaussian is rendered [S1, unofficial].

4.  **Mean Contribution:** The normalizer is the average contribution weight across all pixels $p$ that the Gaussian contributes to in a view:
    $$ \bar{w}_{i,v} = \frac{1}{N_{i,v}} \sum_p w_{i,p} $$
    where $N_{i,v}$ is the number of pixels $p$ where $w_{i,p} > 0$ in view $v$ [S1, unofficial].
    *Note:* The global mean contribution $\bar{w}_i$ used in the final score is typically the sum of per-view means or the mean over all pixels across all views, depending on the specific implementation variant. The core definition uses the per-view mean to avoid footprint bias [S1, unofficial].

5.  **Densification Score:** The final criterion score $\gamma_i$ is:
    $$ \gamma_i = \frac{\|\mathbf{g}_i\|}{1 + \sum_v \bar{w}_{i,v}} $$
    The $+1$ in the denominator prevents division by zero and bounds the score for Gaussians with negligible contribution [S1, unofficial].

**Parameters:**
*   **Threshold $\tau$:** A scalar value triggering densification.
    *   Default: $0.002$ (improves metrics but increases Gaussian count) [S1, unofficial].
    *   Conservative: $0.003$ (reduces count while retaining gains) [S1, unofficial].
    *   Values $> 0.004$ lead to under-densification [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_contribution_normalization_score(gaussian, views):
    """
    Computes the contribution-based normalization score for a single Gaussian.
    
    Args:
        gaussian: Object containing projected mean mu_i, covariance Sigma_i, 
                  opacity alpha, and color c_i.
        views: List of view objects, each containing camera projection info 
               and pre-computed transmittance T for the Gaussian.
               
    Returns:
        score: The densification criterion value.
    """
    accumulated_gradient = [0.0, 0.0] # [x, y] components
    total_mean_contribution = 0.0
    
    for view in views:
        # 1. Compute projected footprint parameters for this view
        # Project 3D mean and covariance to 2D screen space
        mu_2d = project_mean(gaussian.mu_3d, view.camera_matrix)
        Sigma_2d = project_covariance(gaussian.Sigma_3d, view.camera_matrix, gaussian.mu_3d)
        
        # 2. Identify contributing pixels and compute weights
        # In practice, this is done during rasterization. 
        # Here we assume we have access to the list of pixels p where the Gaussian is visible.
        contributing_pixels = get_contributing_pixels(gaussian, view)
        
        view_gradient = [0.0, 0.0]
        sum_weights = 0.0
        count_pixels = 0
        
        for p in contributing_pixels:
            # Compute Gaussian kernel value G_i,p
            diff = p - mu_2d
            exponent = -0.5 * diff.dot(Sigma_2d.inv() * diff)
            G_ip = exp(exponent)
            
            # Compute contribution weight w_i,p = alpha * T
            # alpha is derived from G_ip and the Gaussian's base opacity
            alpha_ip = gaussian.opacity * G_ip 
            T_ip = view.transmittance_at_pixel(p, gaussian) # Pre-computed or computed via depth
            w_ip = alpha_ip * T_ip
            
            # Compute geometric steering term: d(ln(G))/d(mu_x)
            # ln(G) = -0.5 * (p-mu)^T Sigma^-1 (p-mu)
            # d(ln(G))/d(mu) = Sigma^-1 (p - mu)
            steering = Sigma_2d.inv() * diff
            
            # Compute photometric error and contrast term
            # L_p is the loss at pixel p, C_p is the final pixel color
            # dL/dC is the gradient of the loss w.r.t the final pixel color
            dL_dC = view.loss_gradient[p]
            
            # A_i,p is the accumulated color from previous Gaussians (background)
            A_ip = view.accumulated_color_before[p]
            
            contrast = gaussian.color - A_ip
            
            # Gradient component for this pixel
            # g_i,x = dL/dC * (c_i - A_i,p) * w_i,p * steering_x
            pixel_grad = dL_dC * contrast * w_ip * steering
            
            view_gradient[0] += pixel_grad[0]
            view_gradient[1] += pixel_grad[1]
            
            sum_weights += w_ip
            count_pixels += 1
            
        # Compute mean contribution for this view
        if count_pixels > 0:
            mean_w_view = sum_weights / count_pixels
        else:
            mean_w_view = 0.0
            
        total_mean_contribution += mean_w_view
        accumulated_gradient[0] += view_gradient[0]
        accumulated_gradient[1] += view_gradient[1]
        
    # Compute final score
    grad_norm = sqrt(accumulated_gradient[0]**2 + accumulated_gradient[1]**2)
    score = grad_norm / (1.0 + total_mean_contribution)
    
    return score
```

**Edge Cases and Pitfalls:**
*   **Cloning Over-densification:** The clone operation (creating a new Gaussian at the same position) can cause catastrophic over-densification and inflated counts without quality improvement. Mitigation: Disable cloning by overriding the scene extent or forcing split-only densification [S1, unofficial].
*   **Occlusion Bias:** Normalizing by raw view count $M$ (as in standard 3DGS) includes occluded views where the gradient is zero, dragging down the average and stalling densification. This method avoids that by using contribution [S1, unofficial].
*   **Footprint Bias:** Using the per-view total weight $\sum_p w_{i,p}$ as the normalizer reintroduces pixel-count dependence, as larger footprints accumulate more weight. The mean contribution $\bar{w}_i$ is footprint-free [S1, unofficial].

## Subtopics

### Per-view Normalizer
A variant that scales both the gradient sum and the weight sum by the number of rendered views $N_i$ to compute the densification score [S1, unofficial].
**Formula:**
$$ \gamma_i^{\text{per-view}} = \frac{\sum_v \|\mathbf{g}_{i,v}\|/N_i}{1 + \sum_v \bar{w}_{i,v}/N_i} $$
**Comparison:** Produces reconstruction quality similar to the baseline normalizer but requires approximately 1.4× more Gaussians to reach the same operating point [S1, unofficial].

### Sum-Allocation Normalizer
A variant that replaces the per-view mean with the per-view total weight $W_{i,v} = \sum_p w_{i,p}$ in the denominator [S1, unofficial].
**Formula:**
$$ \gamma_i^{\text{sum-alloc}} = \frac{\sum_v \|\mathbf{g}_{i,v}\|}{1 + \sum_v W_{i,v}} $$
**Comparison:** This places the denominator on a different scale and reintroduces footprint dependence, leading to instability or divergence on large scenes compared to the mean contribution variant [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P \cdot V)$, where $N$ is the number of Gaussians, $P$ is the average number of pixels per Gaussian footprint, and $V$ is the number of views. The normalization step is $O(V)$ per Gaussian.
*   **Memory Complexity:** $O(N \cdot V)$ to store accumulated gradients and contribution sums if not computed on the fly.
*   **Performance:** On occlusion-heavy scenes (e.g., Zip-NeRF), this method outperforms standard baselines (like StopThePop) on all image-quality metrics, reducing LPIPS by up to 12% [S1, unofficial]. It keeps the Gaussian count bounded under long training schedules where other criteria diverge [S1, unofficial].
*   **Scaling:** On smaller benchmarks (Mip-NeRF360), it improves metrics at the cost of increased Gaussian count (1.8× on 2DGS, 2.4× on 3DGS at threshold 0.002) [S1, unofficial]. Raising the threshold to 0.003 reduces the count closer to baseline levels while retaining gains [S1, unofficial].
*   **Strengths:** Robust to occlusion; prevents bias toward large footprints; generalizes to 2DGS and 3DGS renderers [S1, unofficial].
*   **Limitations:** Requires careful threshold tuning; prone to over-densification if cloning is not suppressed; higher Gaussian counts than some baseline methods at default thresholds [S1, unofficial].

## Background
Originated in the paper "Counting Contributions, Not Views: Occlusion-Aware Densification for 3D Gaussian Splatting" [S1, unofficial]. The method was developed to address the limitations of view-count normalization in the presence of occlusions, which is common in complex scenes. It builds on the observation that the gradient signal is intrinsically contribution-weighted, making the mean contribution a natural normalizer. The method has been ported and validated on 2DGS and 3DGS frameworks, demonstrating its generalizability beyond the original StopThePop baseline [S1, unofficial].

## Sources
[S1] Counting Contributions, Not Views: Occlusion-Aware Densification for 3D Gaussian Splatting, https://arxiv.org/abs/2610.27108, fetched 2026-10-26 (unofficial)
