# Ray-Centric Adaptive Step and Termination

Also called: Adaptive Ray Marching, Adaptive Step Size Control

## Overview
Ray-Centric Adaptive Step and Termination is a family of volume rendering optimizations that dynamically adjust the sampling density along a ray based on local scene properties to reduce computational cost while preserving visual quality. It distinguishes itself from sibling methods by focusing on spatial adaptation along the ray trajectory (step size) and early exit conditions, rather than temporal frame budgeting [S5, unofficial] or hierarchical preintegration termination [S11].

## Shared principles
The core mechanism involves modifying the standard ray marching loop by dynamically determining the step size $\Delta t$ based on a local metric $m(\mathbf{x})$.

### Inputs and Outputs
*   **Inputs:** Scalar field $f(\mathbf{x})$, ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, transfer function $T(f)$, isovalue $\rho$ (for isosurface rendering), and parameters controlling the adaptation (e.g., thresholds, multipliers).
*   **Outputs:** Final pixel color $C$ and accumulated opacity $\alpha$.

### Data Structures
*   **Metric Storage:** For per-macrocell or per-block adaptation, a scalar metric $m$ is computed for each region [S8].
*   **State Variables:** Accumulated opacity $\alpha$, accumulated color $C$, current position $t$, and step size $\Delta t$ [S6][S9, unofficial].

### Algorithm Steps

1.  **Metric Calculation:** Compute a local metric $m(\mathbf{x})$ to estimate complexity or emptiness. Common metrics include:
    *   **Opacity/Density:** Local scalar value $f(\mathbf{x})$ or segment opacity [S6][S10, unofficial].
    *   **Gradient:** Magnitude $|\nabla f(\mathbf{x})|$ [S6, S7].
    *   **Variance:** Ratio of local cell variance to global variance [S8].
    *   **Null-Region Check:** Difference in accumulated opacity between steps [S10, unofficial].

2.  **Step Size Determination:** Calculate $\Delta t$ based on $m$.
    *   **Inverse Opacity:** $\Delta t \propto 1/f(\mathbf{x})$ or $\Delta t \propto 1/(1-\alpha)$ [S6].
    *   **Gradient-Based:** $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, k / |\nabla f|))$ [S6].
    *   **Variance-Based:** Compute multiplier $M = 2m - 1 + 1$ (where $m = \sqrt{dVar(f_{cell})} / \sqrt{dVar(f_{vol})}$), then $\Delta t_{new} = M \Delta t_{base}$ [S8].
    *   **Null-Region Skip:** If $|\alpha_i - \alpha_{i-1}| < \epsilon$, increase $\Delta t$ significantly [S10, unofficial].

3.  **Quantization for Coherence:** To prevent warp divergence on GPUs, quantize $\Delta t$ to a small set of discrete values (e.g., base rate $r_0$ and max rate $r_1 = n r_0$) [S7, S8].

4.  **Termination Check:** Terminate if accumulated opacity $\alpha > 1 - \epsilon_{term}$ (typically $\epsilon_{term} = 0.01 \dots 0.05$) or ray exits bounding box [S6][S10, unofficial].

5.  **Compositing:** Update color and opacity using front-to-back compositing:
    $$ \alpha_{seg} = 1 - e^{-f(\mathbf{x})\Delta t} $$
    $$ C_{seg} = T(f(\mathbf{x})) \cdot \alpha_{seg} $$
    $$ \alpha \leftarrow \alpha + \alpha_{seg}(1 - \alpha) $$
    $$ C \leftarrow C + C_{seg}(1 - \alpha) $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sample_ray(ray_origin, ray_direction, volume_data, transfer_func, params):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    volume_data: function sample(x) returning scalar f(x)
    transfer_func: function T(f) returning vec3 color
    params: dict containing:
        - base_step: float (initial delta_t)
        - min_step: float
        - max_step: float
        - opacity_threshold: float (1 - epsilon_term)
        - null_threshold: float (epsilon for null region check)
        - gradient_scale: float (k in gradient-based step)
        - discrete_rates: list of floats [r0, r1, ...] for quantization
    """
    t = 0.0
    accumulated_alpha = 0.0
    accumulated_color = vec3(0.0, 0.0, 0.0)
    prev_accumulated_alpha = 0.0
    
    # Determine entry and exit points (simplified, assuming unit sphere or known bounds)
    t_start, t_end = compute_ray_bounds(ray_origin, ray_direction, volume_bounds)
    
    # Initial step size
    step_size = params['base_step']
    
    while t < t_end and accumulated_alpha < params['opacity_threshold']:
        # 1. Sample scalar field
        pos = ray_origin + t * ray_direction
        f_val = volume_data.sample(pos)
        
        # 2. Compute Metric and Adjust Step Size
        # Example: Gradient-based or Opacity-based adaptation
        # Note: Gradient calculation requires neighboring samples or precomputed gradient field
        # Here we use a simplified opacity/null-region check for demonstration
        
        current_segment_opacity = 1.0 - exp(-f_val * step_size)
        
        # Null-region skip logic [S10, unofficial]
        if abs(accumulated_alpha - prev_accumulated_alpha) < params['null_threshold']:
            # Entered null region, increase step size
            step_size = min(step_size * 2.0, params['max_step'])
        else:
            # Reset or refine step size based on local density/gradient
            # Simplified inverse opacity adaptation [S6]
            if f_val > 0:
                step_size = params['gradient_scale'] / f_val
            else:
                step_size = params['max_step']
            
            # Clamp step size
            step_size = clamp(step_size, params['min_step'], params['max_step'])
            
            # Quantize to discrete rates for GPU coherence [S7]
            step_size = quantize_to_discrete_rate(step_size, params['discrete_rates'])
        
        # 3. Compositing
        alpha_seg = 1.0 - exp(-f_val * step_size)
        color_seg = transfer_func(f_val) * alpha_seg
        
        accumulated_alpha += alpha_seg * (1.0 - accumulated_alpha)
        accumulated_color += color_seg * (1.0 - accumulated_alpha)
        
        prev_accumulated_alpha = accumulated_alpha
        t += step_size
        
        # Early termination if fully opaque
        if accumulated_alpha >= params['opacity_threshold']:
            break
            
    return accumulated_color, accumulated_alpha

def quantize_to_discrete_rate(step, rates):
    """
    Selects the closest discrete rate from the list to maintain warp coherence.
    """
    if not rates:
        return step
    min_diff = float('inf')
    best_rate = rates[0]
    for r in rates:
        diff = abs(step - r)
        if diff < min_diff:
            min_diff = diff
            best_rate = r
    return best_rate
```

### Parameters and Typical Values
*   **Base Step Size:** Corresponds to 1 voxel or a fraction thereof [S7].
*   **Opacity Threshold ($\epsilon_{term}$):** $0.01 \dots 0.05$ [S6].
*   **Null Threshold ($\epsilon$):** Small positive value, e.g., $10^{-4}$ [S10, unofficial].
*   **Gradient Scale ($k$):** Tuned to dataset resolution [S6].
*   **Discrete Rates:** $r_0 = 1.0$ (1 voxel), $r_1 = 8 r_0$ [S7].

### Edge Cases and Pitfalls
*   **Warp Divergence:** Fully adaptive per-pixel step sizes cause divergence in GPU warps. Solution: Quantize step sizes to a small set of discrete values [S7, S8].
*   **Missed Intersections:** In isosurface rendering, large steps may skip thin structures. Guaranteed intersections require bounded gradient $L$ and step size $h \le |f - \rho|/L$ [S7].
*   **Texture Cache Thrashing:** Adaptive sampling can lead to poor texture cache utilization if sampling positions diverge significantly between neighboring pixels [S7].

## Subtopics

### [Adaptive Sampling](1385-adaptive-sampling.md)
Adaptive sampling is a volume rendering optimization that varies the sampling density along rays to reduce computational cost while preserving visual quality, by sampling non-homogeneous or high-detail regions more densely than homogeneous or empty regions. Unlike fixed-step methods, it adapts to local data characteristics such as opacity, gradient magnitude, or variance. It is distinguished from its siblings by its primary focus on spatial adaptation along the ray (step size) rather than temporal adaptation (frame budget) or hierarchical preintegration termination.

### [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md)
Adaptive Sampling for Interaction is a performance optimization technique that dynamically lowers the sampling rate during user interaction (such as camera movement or transfer function editing) to maintain interactive frame rates, then restores the high sampling rate once the interaction is complete. Unlike [Adaptive Sampling](1385-adaptive-sampling.md), which varies sampling based on local scene complexity or error estimates, this method varies sampling based on global user input state. It builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by applying a coarse, uniform step size during interaction and a fine, potentially adaptive step size during idle rendering.

### [Adaptive Preintegration Termination Criteria](9804-adaptive-preintegration-termination-criteria.md)
Adaptive Preintegration Termination Criteria is a method for determining when to stop subdividing a ray segment during volume rendering, specifically within the context of adaptive preintegration. It solves the problem of balancing rendering accuracy against computational cost by recursively subdividing ray intervals only when the estimated error exceeds a threshold. It sets itself apart from standard adaptive sampling [1385-adaptive-sampling.md] by operating on preintegrated segment contributions rather than raw scalar samples, and from simple opacity-based termination [5938-ray-centric-adaptive-step-and-termination.md] by using a relative error metric between coarse and fine approximations of the segment integral.

### Adaptive Sampling with Uniform Fallback
- [S1, unofficial] Samples points along rays using adaptive sampling via nerfacc, with a fallback to uniform sampling for rays that receive too few samples.
- [S1, unofficial] Compute t_near and t_far using ray_aabb_intersect. Filter valid rays where t_far > t_near.
- [S1, unofficial] If no valid rays, return empty tensors for t_s, t_e, ray_idx, and valid.
- [S1, unofficial] Call self.nerfacc_estimator.sampling with valid rays_o, rays_d, t_near, t_far, and self.step_size to get ray_idx, t_s, t_e.
- [S1, unofficial] Convert ray_idx to torch.long.
- [S1, unofficial] If self.enable_fallback is True, count samples per ray using torch.bincount(ray_idx, minlength=Nv).
- [S1, unofficial] Identify rays with counts < self.fallback_when_samples as low_ids.
- [S1, unofficial] For low rays, compute uniform samples: t_s_fb = t0[:, None] + k * dt, t_e_fb = t_s_fb + dt, where k is range(K), dt = (t1-t0)/K, and clamp final t_e_fb to t1.
- [S1, unofficial] Construct ray_idx_fb for the fallback samples.
- [S1, unofficial] Concatenate nerfacc samples with fallback samples (implementation detail cut off in text, but implies combining t_s, t_e, ray_idx with t_s_fb, t_e_fb, ray_idx_fb).

### Loss-based Adaptive Sampling
- [S2] A ray sampling strategy that allocates more samples to image regions with higher rendering loss, identified by partitioning the image into an 8x8 grid and computing mean loss per region.
- [S2] Partition each input image into a grid of 64 regions (R_j, j=1..64).
- [S2] Randomly sample a uniform distribution of pixels s_i from the input source views.
- [S2] Normalize the region losses H_i[j] into a probability distribution f_i[j].
- [S2] Resample pixels for the next iteration by allocating n_i * f_i[j] samples to each region R_j, where n_i is the total sample count per image.
- [S2] H_i[j] = \frac{1}{|r_j|} \sum_{(u,v) \in r_j} e_i^g[u, v] + e_i^p[u, v], where r_j = s_i \cap R_j represents pixels uniformly sampled from region R_j, e_i^g is gradient loss, and e_i^p is pixel loss.
- [S2] f_i[j] = \frac{\mathcal{H}_i[j]}{\sum_{m=1}^{64} \mathcal{H}_i[m]}, where the denominator sums the loss statistics over all 64 regions.
- [S2] Grid size: 8x8, resulting in 64 regions (j=1..64).
- [S2] n_i: Total sample count per image used for resampling allocation.

## Background
Adaptive sampling techniques have evolved from fixed-step ray marching to address the inefficiency of sampling empty space or homogeneous regions uniformly. Early implementations focused on opacity-based termination and simple gradient heuristics to skip empty voxels [S6]. The introduction of GPU-based volume rendering highlighted the need for warp coherence, leading to the quantization of step sizes [S7, S8]. More recent developments in neural radiance fields (NeRF) have adapted these principles for loss-based sampling, where sampling density is driven by training error rather than just geometric complexity [S2].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S2] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
