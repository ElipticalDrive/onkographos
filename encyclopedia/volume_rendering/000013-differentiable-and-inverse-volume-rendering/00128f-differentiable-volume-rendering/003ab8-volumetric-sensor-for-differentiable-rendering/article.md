# Volumetric Sensor for Differentiable Rendering
Also called: DeltaVolumetricSensor

## Overview
The Volumetric Sensor (DeltaVolumetricSensor) is a differentiable rendering component that accumulates radiance contributions into a volumetric film using delta tracking logic, supporting both forward and reverse-mode automatic differentiation [S1, unofficial]. It distinguishes itself from standard differentiable volume rendering by explicitly managing the gradient propagation through the scattering and extinction coefficients within the sensor's accumulation step, rather than relying solely on the integrator's path tracing loop [S1, unofficial]. It serves as the final accumulation stage in differentiable volumetric pipelines, bridging the gap between physical simulation and gradient-based optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray parameters (origin, direction, time $t$), volume density field ($\sigma_t$), scattering coefficient ($\sigma_s$), emitted radiance ($em$), and an active ray mask.
*   **Outputs:** Accumulated radiance in the film buffer and gradients for $\sigma_t$, $\sigma_s$, and $em$.

**Data Structures:**
*   **Film Buffer:** Stores accumulated radiance. In primal mode, this is the final output. In forward-mode AD, contributions are scattered into a gradient buffer (`δL.array`).
*   **Gradient Buffers:** `g_em`, `g_st`, `g_ss` initialized to zeros, storing gradients for emitted radiance, total extinction, and scattering coefficients respectively [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize gradient buffers `g_em`, `g_st`, `g_ss` to zeros [S1, unofficial].
2.  **Ray Validation:** Determine active rays using `active = mi.Bool(active & mei.is_valid())` [S1, unofficial].
3.  **Position and Voxel Lookup:** Compute ray position `pos = ray(mei.t)`. Determine the current voxel index `current_voxel` and its flat index `current_voxel_flat`. Check if the position is inside the grid bounds `is_inside_grid` [S1, unofficial].
4.  **Index Selection:** Select the accumulation index. If surface-aware, use `2*current_voxel_flat` for inside and `2*current_voxel_flat+1` for outside. Otherwise, use `current_voxel_flat` [S1, unofficial].
5.  **Gradient Setup:** Detach primal values for `emitted`, `sigma_s`, and `sigma_t` to break the computational graph where necessary. Enable gradients for these variables and set initial gradients using `dr.set_grad` [S1, unofficial].
6.  **Forward-Mode AD Propagation:** If operating in forward-mode AD, propagate gradients from `sigma_t` to `sigma_s` if `sigma_t` has a gradient; otherwise, propagate from `sigma_s`. Set gradients for `ss` and `st` accordingly [S1, unofficial].
7.  **Contribution Calculation:** Compute the radiance contribution using the formulas below.
8.  **Accumulation:**
    *   **Primal Mode:** Write the contribution to the film: `self.m_film.write(contrib, idx, active & is_inside_grid)` [S1, unofficial].
    *   **Forward Mode (Non-Primal):** Scatter and reduce the contribution into the gradient buffer `δL.array` [S1, unofficial].

**Formulas:**
The core accumulation relies on delta tracking statistics [S1, unofficial]:

*   **Scattering Coefficient:**
    $$ sa = \sigma_t - \sigma_s $$
*   **Transmittance:**
    $$ tr = e^{-\sigma_t t} $$
*   **Transmittance PDF:**
    $$ tr_{pdf} = tr \cdot \sigma_t $$
*   **Inverse PDF:**
    $$ inv_{pdf} = \text{select}(tr_{pdf} \neq 0, \text{detach}(1/tr_{pdf}), 0) $$
*   **Throughput:**
    $$ throughput = \text{detach}(attenuation \cdot e^{\sigma_t t_{prev}}) \cdot e^{-\sigma_t t_{prev}} \cdot \text{select}(\sigma_s \neq 0, \sigma_s^{n_{scat}} / \text{detach}(\sigma_s^{n_{scat}}), 1) $$
*   **Contribution:**
    $$ contrib = \text{select}(active, throughput \cdot sa \cdot tr \cdot inv_{pdf} \cdot em, 0) $$

**Parameters:**
*   `n_scat`: Number of scattering events considered in the throughput term. Not given in the sources.
*   `attenuation`: Pre-computed attenuation factor. Not given in the sources.
*   `t_prev`: Previous time/distance step. Not given in the sources.

**Pitfalls:**
*   **Inverse PDF Calculation:** The calculation of `inv_pdf` is noted as potentially incorrect if free-flight distances are not sampled proportional to transmittance [S1, unofficial]. Implementers must ensure the sampling strategy matches the PDF definition to avoid biased gradients.
*   **Gradient Detachment:** Care must be taken when using `detach()` in the throughput and inverse PDF calculations to prevent gradient flow through terms that should remain constant during the specific AD mode [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import drjit as dr
import mitsuba as mi

def volumetric_sensor_step(ray, mei, sigma_t, sigma_s, em, active, is_inside_grid, 
                           current_voxel_flat, surface_aware=False, mode='primal'):
    """
    Computes radiance contribution and handles gradient accumulation for the Volumetric Sensor.
    
    Args:
        ray: Ray object providing position and time.
        mei: Microstate or execution context.
        sigma_t: Total extinction coefficient tensor.
        sigma_s: Scattering coefficient tensor.
        em: Emitted radiance tensor.
        active: Boolean tensor of active rays.
        is_inside_grid: Boolean tensor indicating if ray is inside volume bounds.
        current_voxel_flat: Flat index of the current voxel.
        surface_aware: Boolean flag for surface-aware accumulation indexing.
        mode: 'primal' or 'forward'.
        
    Returns:
        contrib: Radiance contribution tensor.
    """
    # 1. Initialize gradients (Assumed handled by framework or explicit zeroing)
    # dr.set_grad(sigma_t, dr.zeros_like(sigma_t))
    # dr.set_grad(sigma_s, dr.zeros_like(sigma_s))
    # dr.set_grad(em, dr.zeros_like(em))
    
    # 2. Determine active rays
    # active = mi.Bool(active & mei.is_valid()) 
    # Note: 'mei.is_valid()' is context-dependent; assuming 'active' is pre-filtered
    
    # 3. Compute ray position and voxel index
    # pos = ray(mei.t) 
    # current_voxel_flat is provided as input
    
    # 4. Select index for accumulation
    if surface_aware:
        # Inside uses 2*idx, Outside uses 2*idx + 1
        # Assuming 'is_inside_grid' is a boolean tensor
        idx = dr.select(is_inside_grid, 2 * current_voxel_flat, 2 * current_voxel_flat + 1)
    else:
        idx = current_voxel_flat
        
    # 5. Detach and enable gradients for physical quantities
    # Detach primal values to control gradient flow
    em_detached = dr.detach(em)
    sigma_s_detached = dr.detach(sigma_s)
    sigma_t_detached = dr.detach(sigma_t)
    
    # Enable gradients (Framework specific, e.g., dr.enable_grad)
    # dr.enable_grad(sigma_t)
    # dr.enable_grad(sigma_s)
    # dr.enable_grad(em)
    
    # 6. Forward-mode AD Propagation
    if mode == 'forward':
        # Propagate gradients from sigma_t to sigma_s if sigma_t has gradient, else from sigma_s
        # This logic depends on the specific AD engine's gradient state
        if dr.has_grad(sigma_t):
            # Propagate from sigma_t to sigma_s
            # dr.set_grad(sigma_s, dr.grad(sigma_t)) # Conceptual
            pass
        elif dr.has_grad(sigma_s):
            # Propagate from sigma_s
            pass
            
        # Set gradients for ss and st (Context specific)
        # dr.set_grad(...)
        
    # 7. Compute Physics Terms
    # Scattering coefficient
    sa = sigma_t - sigma_s
    
    # Transmittance
    # t is assumed to be the step size or distance to next interaction
    # For this snippet, assuming 't' is passed or derived from ray
    # t = ... 
    tr = dr.exp(-sigma_t_detached * 1.0) # Placeholder for t=1 or specific step
    
    # Transmittance PDF
    tr_pdf = tr * sigma_t_detached
    
    # Inverse PDF
    # select(condition, true_val, false_val)
    inv_pdf = dr.select(tr_pdf != 0, dr.detach(1.0 / tr_pdf), 0.0)
    
    # Throughput
    # attenuation, t_prev, n_scat are assumed available in context
    # throughput = dr.detach(attenuation * dr.exp(sigma_t_detached * t_prev)) * dr.exp(-sigma_t_detached * t_prev) * ...
    # Simplified placeholder for throughput calculation
    throughput = 1.0 
    
    # 8. Contribution
    contrib = dr.select(active, throughput * sa * tr * inv_pdf * em_detached, 0.0)
    
    # 9. Accumulation
    if mode == 'primal':
        # self.m_film.write(contrib, idx, active & is_inside_grid)
        pass
    else:
        # Scatter reduce contribution into δL.array
        # δL.array.scatter_add(idx, contrib)
        pass
        
    return contrib
```

## Subtopics

### RatioVolumetricSensor
A variant of the volumetric sensor that uses a majorant for ratio tracking or similar variance reduction techniques in volumetric rendering [S1, unofficial]. It requires a `majorant` property during initialization, which represents the maximum extinction coefficient used to bound the sampling probability [S1, unofficial]. This variant is chosen when variance in the gradient estimation is high due to heterogeneous media, trading off some computational overhead for improved convergence stability.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active rays and samples per ray, dominated by the exponential and selection operations in the contribution formula [S1, unofficial].
*   **Memory Complexity:** $O(n)$ for the film buffer and gradient buffers, where $n$ is the number of voxels or pixels, depending on the accumulation target [S1, unofficial].
*   **Strengths:** Supports both forward and reverse-mode AD, allowing flexibility in optimization strategies. Explicit gradient handling can reduce memory overhead compared to full reverse-mode AD through the entire integrator [S1, unofficial].
*   **Limitations:** The inverse PDF calculation is sensitive to sampling strategies; incorrect sampling proportional to transmittance leads to biased gradients [S1, unofficial]. Requires specific Mitsuba variants (`cuda_ad_mono`, `llvm_ad_mono`, `metal_ad_mono`) to function [S1, unofficial].
*   **When to use:** When optimizing volumetric parameters (density, scattering) directly via gradients and when memory constraints prohibit storing the full computational graph for reverse-mode AD [S1, unofficial].
*   **When not to use:** If the sampling strategy cannot be guaranteed to match the transmittance distribution, or if the environment does not support the required Dr.Jit/Mitsuba variants [S1, unofficial].

## Background
The Volumetric Sensor is part of the Dr. TVAM (Tomographic Volumetric Additive Manufacturing) project, an inverse rendering framework for tomographic volumetric additive manufacturing [S1, unofficial]. It was developed to enable gradient-based optimization of volumetric properties in manufacturing contexts, leveraging the Dr.Jit library for differentiable programming [S1, unofficial]. The implementation is tied to specific variants of the Mitsuba renderer that support automatic differentiation [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
