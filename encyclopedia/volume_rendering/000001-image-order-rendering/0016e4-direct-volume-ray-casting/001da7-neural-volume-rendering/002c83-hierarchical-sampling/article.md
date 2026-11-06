# Hierarchical Sampling
Also called: Coarse-Fine Sampling, Coarse-to-Fine Sampling, Importance Sampling for NeRF, Two-Stage Sampling

## Overview
Hierarchical sampling is a two-stage sampling strategy for direct volume ray casting that uses a coarse estimate of the scene's geometry or density to guide the placement of fine samples via importance sampling [S1, S2, S4, unofficial]. It solves the problem of inefficient uniform sampling by concentrating computational effort in regions of high opacity or high-frequency detail, thereby reducing aliasing and noise while maintaining image quality [S2, unofficial][S3][S4, unofficial]. Unlike uniform sampling, which treats all segments of a ray equally, hierarchical sampling adapts the sample density based on the estimated contribution of each segment, often achieving 2-3x efficiency gains over uniform sampling without visible artifacts [S3].

## How it works
The method operates in two distinct passes: a coarse pass to estimate the density distribution along the ray, and a fine pass to render the final color using samples drawn from that distribution.

**Inputs and Outputs:**
*   **Input:** Camera ray $r(t) = O + tD$, bounds $[t_{start}, t_{end}]$, coarse network/model $N_c$, fine network/model $N_f$ (or transfer function), number of coarse samples $n_{coarse}$, number of fine samples $n_{fine}$.
*   **Output:** Final pixel color $C$ and accumulated opacity $\alpha$.

**Data Structures:**
*   **Coarse Samples:** A set of depths $T_c = \{t_{c,1}, ..., t_{c,n_{coarse}}\}$ and associated weights $W_c = \{w_{c,1}, ..., w_{c,n_{coarse}}\}$.
*   **Fine Samples:** A set of depths $T_f = \{t_{f,1}, ..., t_{f,n_{fine}}\}$ derived from importance sampling.
*   **CDF:** A cumulative distribution function constructed from $W_c$.

**Steps:**

1.  **Coarse Pass:**
    *   Generate $n_{coarse}$ sample points $t_{c,i}$ uniformly or stratified along the ray segment $[t_{start}, t_{end}]$ [S2, unofficial][S3].
    *   Evaluate the coarse model $N_c$ at each $t_{c,i}$ to obtain density $\sigma_c(t_{c,i})$ and color $C_c(t_{c,i})$ [S1, S2, S4, unofficial].
    *   Compute the weight $w_{c,i}$ for each sample. In the context of volume rendering, the weight represents the expected contribution of the sample. For a discretized volume rendering integral, the weight is often proportional to the opacity contribution:
        $$ w_{c,i} = \alpha_{c,i} \prod_{j=1}^{i-1} (1 - \alpha_{c,j}) $$
        where $\alpha_{c,i}$ is the opacity derived from $\sigma_c(t_{c,i})$ and the step size $\Delta t$ [S2, unofficial].
    *   Normalize the weights to form a probability distribution:
        $$ p_{c,i} = \frac{w_{c,i}}{\sum_{k=1}^{n_{coarse}} w_{c,k}} $$

2.  **Importance Sampling (Fine Sample Generation):**
    *   Construct the Cumulative Distribution Function (CDF) from the normalized weights $p_{c,i}$ [S2, unofficial].
    *   Draw $n_{fine}$ random numbers $u_k \sim \text{Uniform}(0, 1)$ for $k=1, ..., n_{fine}$ [S2, unofficial].
    *   Map each $u_k$ to a depth $t_{f,k}$ using the inverse CDF of the coarse distribution. This places fine samples in regions where the coarse model predicted high density or opacity [S2, S4, unofficial].
    *   *Note:* The sampling step is non-differentiable with respect to the coarse model's output if gradients are intended to flow back to update the coarse model. Implementations typically detach the inputs to the sampling function to prevent gradient flow from the fine pass to the coarse pass [S2, unofficial].

3.  **Fine Pass:**
    *   Evaluate the fine model $N_f$ at each $t_{f,k}$ to obtain density $\sigma_f(t_{f,k})$ and color $C_f(t_{f,k})$ [S1, S2, S4, unofficial].
    *   Perform front-to-back compositing using the fine samples $T_f$ and their corresponding colors and opacities to compute the final pixel color $C$ and opacity $\alpha$ [S2, unofficial][S3][S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def hierarchical_sample(ray_origin, ray_direction, t_start, t_end, 
                        n_coarse, n_fine, coarse_model, fine_model, 
                        transfer_function):
    """
    Performs hierarchical sampling for volume rendering.
    
    Args:
        ray_origin: (3,) array, ray origin O.
        ray_direction: (3,) array, ray direction D (normalized).
        t_start: float, near bound.
        t_end: float, far bound.
        n_coarse: int, number of coarse samples.
        n_fine: int, number of fine samples.
        coarse_model: function, returns (density, color) given ray position.
        fine_model: function, returns (density, color) given ray position.
        transfer_function: function, maps density to (color, opacity).
        
    Returns:
        final_color: (3,) array, rendered color.
        final_opacity: float, accumulated opacity.
    """
    
    # --- Coarse Pass ---
    # 1. Generate coarse sample depths (stratified sampling)
    coarse_depths = []
    for i in range(n_coarse):
        # Stratified sample within [t_start, t_end]
        u = (i + random.uniform(0, 1)) / n_coarse
        t = t_start + u * (t_end - t_start)
        coarse_depths.append(t)
    
    # 2. Evaluate coarse model and compute weights
    coarse_weights = []
    coarse_opacities = []
    for t in coarse_depths:
        pos = ray_origin + t * ray_direction
        density, _ = coarse_model(pos) # Coarse model returns density and maybe color
        color, opacity = transfer_function(density)
        coarse_opacities.append(opacity)
        # Weight is opacity contribution. 
        # Note: In strict NeRF implementations, weights are often just the 
        # opacity term alpha_i * product(1-alpha_j). Here we simplify to 
        # just opacity for importance sampling if step size is uniform, 
        # or compute full contribution if step sizes vary.
        # For uniform step size dt, weight ~ opacity.
        coarse_weights.append(opacity)
    
    # 3. Normalize weights to form probability distribution
    total_weight = sum(coarse_weights)
    if total_weight == 0:
        # Fallback to uniform sampling if no density detected
        probs = [1.0 / n_coarse] * n_coarse
    else:
        probs = [w / total_weight for w in coarse_weights]
    
    # 4. Construct CDF
    cdf = []
    cumulative = 0.0
    for p in probs:
        cumulative += p
        cdf.append(cumulative)
    
    # 5. Sample fine depths using inverse CDF (Importance Sampling)
    fine_depths = []
    for _ in range(n_fine):
        u = random.uniform(0, 1)
        # Find index in CDF
        idx = 0
        while idx < len(cdf) - 1 and cdf[idx] < u:
            idx += 1
        
        # Interpolate between coarse depths for smoother sampling
        # Linear interpolation between t_c[idx-1] and t_c[idx]
        if idx == 0:
            t_fine = coarse_depths[0]
        else:
            # Map u to the interval [cdf[idx-1], cdf[idx]]
            # This is a simplified inverse CDF lookup. 
            # For exact implementation, one would interpolate the depth 
            # corresponding to u within the CDF segment.
            # Here we just pick the coarse depth associated with the bin.
            t_fine = coarse_depths[idx]
        
        fine_depths.append(t_fine)
    
    # --- Fine Pass ---
    # 1. Evaluate fine model at fine depths
    fine_samples = []
    for t in fine_depths:
        pos = ray_origin + t * ray_direction
        density, _ = fine_model(pos)
        color, opacity = transfer_function(density)
        fine_samples.append((t, color, opacity))
    
    # 2. Sort fine samples by depth (front-to-back)
    fine_samples.sort(key=lambda x: x[0])
    
    # 3. Front-to-back compositing
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    
    for t, color, opacity in fine_samples:
        # Standard front-to-back compositing
        # C' = C' + (1 - alpha') * alpha * C
        # alpha' = alpha' + (1 - alpha') * alpha
        weight = (1.0 - accumulated_opacity) * opacity
        accumulated_color[0] += weight * color[0]
        accumulated_color[1] += weight * color[1]
        accumulated_color[2] += weight * color[2]
        accumulated_opacity += weight
        
        # Early ray termination
        if accumulated_opacity >= 0.99:
            break
            
    return accumulated_color, accumulated_opacity
```

**Parameters:**
*   $n_{coarse}$: Typically 16-64 samples [S2, unofficial].
*   $n_{fine}$: Typically 48-128 samples [S2, unofficial].
*   Threshold for early termination: 0.99 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-differentiability:** The sampling step is non-differentiable with respect to the coarse model's output. If gradients are needed for the coarse model, the inputs to the sampling function must be detached [S2, unofficial].
*   **Zero Density:** If the coarse model predicts zero density everywhere, the weights are zero, leading to division by zero in normalization. A fallback to uniform sampling is required [G].
*   **Boundary Conditions:** Ensure that fine samples generated via inverse CDF remain within $[t_{start}, t_{end}]$ [G].

## Subtopics

### [Hierarchical Density Adaptation](4868-hierarchical-density-adaptation.md)
A variant that derives coarse density by convolving a dense feature grid with a non-learnable kernel, avoiding dedicated coarse grid allocation.

## Operating characteristics
*   **Time Complexity:** $O(n_{coarse} + n_{fine})$ per ray. The overhead of constructing the CDF is $O(n_{coarse})$.
*   **Memory Complexity:** $O(n_{coarse})$ to store coarse weights and CDF.
*   **Performance:** Achieves 2-3x efficiency gains over uniform sampling in boundary-heavy volumes like CT angiography data, without aliasing artifacts [S3].
*   **Strengths:** Reduces noise and aliasing in high-frequency regions; efficient use of samples in smooth regions.
*   **Limitations:** Requires two model evaluations (coarse and fine); non-differentiable sampling complicates training if joint optimization is desired.

## Background
Hierarchical sampling was introduced in the context of Neural Radiance Fields (NeRF) to address the inefficiency of uniform sampling in sparse and complex scenes [S1, S2, S4, unofficial]. It builds on the volume rendering integral discretization [S3, S5, unofficial]. The method was popularized by the original NeRF paper and subsequent implementations [S2, S4, unofficial]. It is widely used in neural volume rendering and direct volume ray casting for medical imaging and geophysical data [S3].

## Sources
[S1] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S2] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S3] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S4] Volume Rendering Integral: Definition & AI/ML Use, https://inferensys.com/glossary/neural-radiance-fields-and-spatial-computing/neural-scene-representations/volume-rendering-integral, fetched 2026-10-28 (unofficial)
