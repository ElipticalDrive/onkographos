# Volumetric Primitive Tomography Integrator

Also called: Volumetric Tomography Integrator

## Overview
The Volumetric Primitive Tomography Integrator is a simplified rendering method for volumetric primitives that accounts exclusively for absorption, ignoring scattering and emission [S1, unofficial]. It is designed for tomography applications where the goal is to visualize density or attenuation rather than photometric appearance [S1, unofficial]. Unlike its sibling [Volumetric Primitive PRB Integrator](2543-volumetric-primitive-prb-integrator.md), which handles scattering media, this integrator treats the volume as a purely absorbing medium [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $o$, direction $d$, scene geometry (volumetric primitives), and differentiable rendering backend state.
*   **Output:** Radiance accumulator $L$ (primal mode) or differential radiance $\delta L$ (adjoint mode).

**Data Structures:**
*   `active`: Boolean mask indicating which rays are currently interacting with primitives.
*   `depth`: Integer counter for recursion depth per ray.
*   `L`: Accumulator for radiance.
*   `β`: Path throughput (transmission product).
*   `si`: Surface intersection state containing primitive index, position, and shape data.

**Parameters:**
*   `max_depth` (int): Maximum path depth. `-1` indicates no limit, mapped internally to `0xFFFFFFFF` [S1, unofficial].
*   `kernel_type` (str): Kernel function for density evaluation, one of `['gaussian', 'epanechnikov']` [S1, unofficial].
*   `hide_emitters` (bool): If true, environment emitters are excluded from indirect light sampling [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize primal mode, ray, active mask, depth counter, radiance accumulator $L$, differential radiance $\delta L$, and path throughput $\beta$ [S1, unofficial].
2.  **Intersection Loop:** While `active` is true:
    a. Intersect ray with scene using `scene.ray_intersect` with flags `All | BackfaceCulling` and coherent intersection for depth 0 [S1, unofficial].
    b. **Transmission Evaluation:**
        i. Gather ellipsoid data: `ellipsoid = Ellipsoid.gather(si.shape, si.prim_index, active)` [S1, unofficial].
        ii. Get total extinction coefficient: `sigmat = si.shape.eval_attribute_1('sigma_t', si, active)` [S1, unofficial].
        iii. Compute density integral along ray: `density = self.kernel.density_integral(ray, ellipsoid, None, None, active)` [S1, unofficial].
        iv. Calculate transmission: $T = \exp(-\text{density} \times \sigma_t)$ [S1, unofficial].
    c. **Update Throughput:** $\beta[\text{active}] \ *= T$ [S1, unofficial].
    d. **Adjoint Indirect Radiance:** If in adjoint mode, compute differentiable reflected indirect radiance: $L_{r\_ind} = L \times T / \text{dr.detach}(T)$ [S1, unofficial].
    e. **Local Contribution:** Compute total local contribution: $L_o = L_e + L_{r\_ind}$ (where $L_e = 0$). Select finite values: $L_o = \text{dr.select}(\text{active} \ \& \ \text{dr.isfinite}(L_o), L_o, 0.0)$ [S1, unofficial].
    f. **Gradient Propagation:**
        i. If Backward: `dr.backward_from(δL * L_o)` [S1, unofficial].
        ii. If Forward: `δL += dr.forward_to(L_o)` [S1, unofficial].
    g. **Escape Check:** Check for escape from medium: `escaped_medium = active & ~si.is_valid()`. Exclude environment emitters if `hide_emitters` is true and depth is 0 [S1, unofficial].
    h. **Direct Light:** Compute direct light from environment: $L_{dir} = \text{dr.select}(\text{escaped\_medium}, \beta \times \text{emitter.eval}(si, \text{escaped\_medium}), 0.0)$ [S1, unofficial].
    i. **Update Accumulator:** Update radiance accumulator $L$:
        $$ L[\text{active}] = \begin{cases} L + L_{dir} & \text{if primal} \\ L - L_{dir} & \text{if adjoint} \end{cases} $$
        [S1, unofficial].
    j. **Ray Offset:** Spawn new ray origin to avoid self-intersection: $o[\text{active}] = p + d \times 10^{-4}$ [S1, unofficial].
    k. **Update State:** Update active mask and depth: `active &= si.is_valid()` and `depth[active] += 1` [S1, unofficial].
    l. **Depth Limit:** Limit recursion depth: `active &= depth < self.max_depth` [S1, unofficial].
3.  **Return:** Return $L$ if primal, else $\delta L$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def integrate_ray(ray, scene, active, depth, L, beta, dr_mode):
    # Initialization handled by caller or setup phase
    while active.any():
        # Intersect with scene
        si = scene.ray_intersect(ray, flags=All | BackfaceCulling, depth=0)
        
        # Evaluate transmission for active primitives
        ellipsoid = Ellipsoid.gather(si.shape, si.prim_index, active)
        sigmat = si.shape.eval_attribute_1('sigma_t', si, active)
        density = kernel.density_integral(ray, ellipsoid, None, None, active)
        transmission = exp(-density * sigmat)
        
        # Update path throughput
        beta[active] *= transmission
        
        if dr_mode == 'adjoint':
            # Compute differentiable reflected indirect radiance
            Lr_ind = L * transmission / dr.detach(transmission)
            Lo = Lr_ind # Le is 0 for tomography
            Lo = dr.select(active & dr.isfinite(Lo), Lo, 0.0)
            
            # Propagate gradients
            dr.backward_from(Lo)
        else:
            Lo = 0.0 # No emission or scattering contribution in primal tomography
            
        # Check for escape from medium
        escaped_medium = active & ~si.is_valid()
        
        # Compute direct light from environment
        if escaped_medium.any():
            L_dir = beta * scene.emitter.eval(si, escaped_medium)
        else:
            L_dir = 0.0
            
        # Update radiance accumulator
        if dr_mode == 'primal':
            L[active] = L[active] + L_dir
        else:
            L[active] = L[active] - L_dir
            
        # Spawn new ray origin to avoid self-intersection
        ray.o[active] = si.p + ray.d * 1e-4
        
        # Update active mask and depth
        active = active & si.is_valid()
        depth[active] += 1
        
        # Limit recursion depth
        if max_depth != -1:
            active = active & (depth < max_depth)
            
    return L if dr_mode == 'primal' else dr.get_grad(L)
```

## Subtopics

### Transmission Evaluation
Evaluates the transmission model on intersected volumetric primitives for the tomography integrator [S1, unofficial]. It gathers ellipsoid data, retrieves the total extinction coefficient $\sigma_t$, and computes the density integral along the ray using the specified kernel (Gaussian or Epanechnikov) [S1, unofficial]. The final transmission is calculated as $T = \exp(-\text{density} \times \sigma_t)$ [S1, unofficial]. This step is critical for determining the attenuation of light as it passes through the volumetric primitives [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The complexity is dominated by the ray-primitive intersection and the density integral evaluation per active ray per step [S1, unofficial]. Memory usage is proportional to the number of active rays and the storage for primitive attributes [S1, unofficial].

**Measured Performance:**
Specific performance metrics (FPS, memory usage) are not provided in the sources [S1, unofficial]. The implementation uses Dr.Jit for differentiable rendering, which allows for efficient GPU execution [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Simple and efficient for tomography applications where only absorption matters [S1, unofficial]. Differentiable, allowing for gradient-based optimization [S1, unofficial].
*   **Limitations:** Does not account for scattering or emission, making it unsuitable for general volume rendering or scattering media [S1, unofficial].

**When to Use:**
Use this integrator when rendering volumetric primitives for tomography or attenuation-based visualization [S1, unofficial]. Do not use it for scenes requiring scattering or emission effects [S1, unofficial].

## Background
The Volumetric Primitive Tomography Integrator is part of the `volumetric_primitives` repository by Facebook Research [S1, unofficial]. It is designed to work with 3D Gaussian Splatting-like assets and other volumetric primitive representations [S1, unofficial]. The integrator is one of several provided for different applications, including the VPRF integrator for 3D Gaussian Splatting and the VPPT integrator for scattering media [S1, unofficial].

## Sources
[S1] facebookresearch/volumetric_primitives, https://github.com/facebookresearch/volumetric_primitives, fetched 2026-10-26 (unofficial)
