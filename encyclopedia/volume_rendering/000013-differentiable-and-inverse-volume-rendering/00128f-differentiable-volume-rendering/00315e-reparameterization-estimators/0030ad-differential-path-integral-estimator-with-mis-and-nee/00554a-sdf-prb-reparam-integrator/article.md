# SDF Prb Reparam Integrator

## Overview
A differentiable path tracer that computes gradients of radiance with respect to scene parameters (e.g., Signed Distance Function shape parameters) by warping random samples via reparameterization, supporting both forward and backward mode automatic differentiation [S1, unofficial]. It distinguishes itself from sibling methods by explicitly integrating reparameterization gradients into the path sampling loop, allowing for stable gradient flow through the SDF evaluation and sphere tracing steps, whereas siblings like [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md) rely on vertex perturbation and [Differential Edge Sampling](6708-differential-edge-sampling.md) on edge perturbations.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene description containing an SDF field, camera parameters, random number generator state, and scene parameters $\theta$ (e.g., SDF coefficients).
*   **Outputs:** Primal radiance image $L$ and gradient images $\nabla_\theta L$ (forward or backward).

**Data Structures:**
*   `self.primal_image`: Tensor storing the computed radiance.
*   `self.warp_field`: Optional field defining the reparameterization warp.
*   `reparam_max_depth`: Integer, maximum depth for reparameterization rays.
*   `shade_gradient_max_depth`: Integer, maximum depth for shading gradients, set equal to `reparam_max_depth` if `warp_field` is not None.
*   `reparam_rays`: Integer, number of reparameterization rays (default 16).
*   `reparam_kappa`: Float, reparameterization parameter (default $10^5$).
*   `reparam_exp`: Float, reparameterization exponent (default 3.0).
*   `rr_depth`: Integer, Russian Roulette depth (default 5).
*   `use_aovs`: Boolean, Auxiliary Output Variables flag (default False).
*   `hide_emitters`: Boolean, hide emitters flag (default False).

**Core Algorithm:**
The integrator operates in three phases: Primal Setup, Forward Gradient, and Backward Gradient. It utilizes a "Path Replay Backpropagation" loop when reparameterization is active.

1.  **Primal Setup:**
    *   Prepare the scene SDF via `self.prepare_sdf(scene)`.
    *   Suspend gradients for primal setup using `dr.suspend_grad()` to avoid tracking gradients during static scene preparation [S1, unofficial].
    *   Prepare the sampler and generate primary rays. If `self.warp_field` is not None, apply reparameterization warping to the random samples [S1, unofficial].
    *   Compute primal radiance $L$ by calling `self.sample` with `mode=dr.ADMode.Primal`.
    *   Store $L$ in `self.primal_image`.

2.  **Forward-Mode Gradients:**
    *   Resume gradients.
    *   Compute sample position derivatives if enabled.
    *   Compute forward-mode gradients by calling `self.sample` with `mode=dr.ADMode.Forward`.
    *   Accumulate the reparameterization result gradients into `result_grad` [S1, unofficial].

3.  **Backward-Mode Gradients:**
    *   Prepare the scene SDF via `self.prepare_sdf(scene)`.
    *   Suspend gradients for primal setup using `dr.suspend_grad()`.
    *   Generate primary rays with reparameterization.
    *   Compute primal radiance $L$ using `self.sample` with `mode=dr.ADMode.Primal`.
    *   Resume gradients and enable gradient tracking on $L$.
    *   Set the gradient of the image to `grad_in` and enqueue the backward pass.
    *   Compute backward-mode gradients by calling `self.sample` with `mode=dr.ADMode.Backward` [S1, unofficial].

**Sampling Loop (`self.sample`):**
The sampling loop handles path construction and reparameterization integration.

1.  Initialize loop state: `depth=0`, `L=0` (or `state_in`), $\beta=1$, $\eta=1$, `mis_em=1`.
2.  Perform preliminary ray intersection to get `pi_cur`.
3.  If `warp_field` is not None:
    *   Update `reparam_max_depth` to `max_reparam_depth` if `>= 0`, else $100,000$.
    *   Set `shade_gradient_max_depth` equal to `reparam_max_depth`.
4.  If reparameterization requires `DRJIT_STRUCT`, start a loop named 'Path Replay Backpropagation' [S1, unofficial].
5.  **Sphere Tracing Step (for SDF evaluation):**
    *   Evaluate SDF value $s = \text{SDF}(p)$ and gradient $g = \nabla \text{SDF}(p)$.
    *   Compute silhouette weight $w_{sil}$:
        $$ w_{sil} = \frac{1}{(10^{-7} + |s| + \text{SIL\_WEIGHT\_OFFSET} \cdot (n \cdot d)^2)^3} $$
        where $n = g / \|g\|$, $d$ is the ray direction, and $\text{SIL\_WEIGHT\_OFFSET} = 0.05$.
    *   Compute segment length and update integrals for warp position estimation using the trapezoid rule:
        $$ W_{int} \leftarrow W_{int} + 0.5 \cdot \Delta t \cdot (w_{prev} + w_{sil}) $$
        $$ T_{warp} \leftarrow T_{warp} + 0.5 \cdot \Delta t \cdot (t_{prev} \cdot w_{prev} + t \cdot w_{sil}) $$
    *   Update ray position: $t \leftarrow t + s$.
    *   Check for intersection ($|s| < \epsilon$) and boundary conditions.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import drjit as dr
import mitsuba as mi

def render_forward(scene, params):
    # 1. Primal Setup
    with dr.suspend_grad():
        sdf = prepare_sdf(scene)
        sampler = prepare_sampler()
        # Generate primary rays with reparameterization if warp_field is active
        rays = generate_primary_rays(sampler, warp_field=scene.warp_field)
        
        # Compute primal radiance
        primal_image = sample(scene, rays, mode=dr.ADMode.Primal, sdf=sdf)
        
    # 2. Forward Gradients
    dr.resume_grad()
    # Compute sample position derivatives if enabled
    # [G] Ensure gradients are attached to params before calling sample
    
    # Compute forward-mode gradients
    grad_image = sample(scene, rays, mode=dr.ADMode.Forward, sdf=sdf)
    
    # Accumulate reparameterization result gradients
    result_grad = accumulate_gradients(grad_image)
    return primal_image, result_grad

def render_backward(scene, params, grad_in):
    # 1. Primal Setup
    with dr.suspend_grad():
        sdf = prepare_sdf(scene)
        sampler = prepare_sampler()
        # Generate primary rays with reparameterization
        rays = generate_primary_rays(sampler, warp_field=scene.warp_field)
        
        # Compute primal radiance
        primal_image = sample(scene, rays, mode=dr.ADMode.Primal, sdf=sdf)
        
    # 2. Backward Gradients
    dr.resume_grad()
    dr.enable_grad(primal_image)
    
    # Set gradient of image to grad_in
    primal_image.grad = grad_in
    
    # Enqueue backward pass and compute backward-mode gradients
    grad_image = sample(scene, rays, mode=dr.ADMode.Backward, sdf=sdf)
    
    return grad_image

def sample(scene, rays, mode, sdf):
    """
    Core sampling loop supporting Path Replay Backpropagation.
    """
    depth = 0
    L = 0.0
    beta = 1.0
    eta = 1.0
    mis_em = 1.0
    
    # Initialize reparameterization state if needed
    if scene.warp_field is not None:
        reparam_max_depth = scene.max_reparam_depth if scene.max_reparam_depth >= 0 else 100000
        shade_gradient_max_depth = reparam_max_depth
    else:
        reparam_max_depth = 0
        shade_gradient_max_depth = 0
        
    # Path Replay Backpropagation Loop
    # [G] This loop structure is specific to DRJIT/Adjoint mode handling
    # The loop continues until max depth or termination condition
    while depth < shade_gradient_max_depth:
        # Preliminary ray intersection
        pi_cur = intersect_rays(rays, scene)
        
        if pi_cur.hit:
            # Sphere tracing step for SDF
            p = pi_cur.position
            sdf_val = sdf.eval(p)
            sdf_grad = sdf.eval_grad(p)
            
            # Compute silhouette weight
            n = sdf_grad / dr.norm(sdf_grad)
            n_dot_d = dr.dot(rays.d, n)
            sil_weight_offset = 0.05
            w_sil = 1.0 / (1e-7 + dr.abs(sdf_val) + sil_weight_offset * n_dot_d**2)**3
            w_sil = dr.maximum(w_sil, 0.0)
            
            # Update reparameterization integrals (Trapezoid rule)
            # [G] Assuming state variables for weight_integral and warp_t_integral are maintained
            # in the loop context or passed as state
            segment_length = dr.maximum(prev_sdf_value, 0.0)
            prev_t = t - prev_sdf_value
            
            weight_integral += 0.5 * segment_length * (prev_sil_w + w_sil)
            warp_t_integral += 0.5 * segment_length * (prev_t * prev_sil_w + t * w_sil)
            
            # Update ray position
            t += sdf_val
            prev_sdf_value = sdf_val
            prev_sil_w = w_sil
            depth += 1
            
            # Check for termination
            if dr.abs(sdf_val) < 1e-6:
                # Surface hit, compute BSDF contribution
                # [G] Standard path tracing BSDF evaluation and MIS weighting
                L += beta * compute_radiance(scene, rays, sdf_val, mis_em)
                break
        else:
            # No hit, terminate or continue
            break
            
    return L
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the maximum path depth. The reparameterization step adds overhead proportional to the number of reparameterization rays (`reparam_rays`, default 16) and the cost of evaluating the SDF and its gradient at each step.
*   **Memory Complexity:** Linear with respect to the image resolution and path depth. The "Path Replay Backpropagation" requires storing path history for the backward pass, increasing memory usage compared to primal-only rendering.
*   **Performance:** Performance depends heavily on the complexity of the SDF evaluation. The use of `reparam_rays=16` provides a balance between gradient accuracy and computational cost.
*   **Strengths:** Provides stable gradients for SDF-based shape optimization. Supports both forward and backward mode AD, allowing flexibility in optimization setups.
*   **Limitations:** Requires the SDF to be differentiable. The reparameterization warping may introduce bias if the warp field is poorly conditioned. High `reparam_kappa` ($10^5$) and `reparam_exp` (3.0) are used to stabilize the warping but may require tuning for specific scenes.
*   **When to use:** When optimizing SDF parameters for shape reconstruction or editing.
*   **When not to use:** When the scene does not involve SDFs or when memory is extremely constrained (due to path replay storage).

## Background
The method originates from the work on Differentiable Signed Distance Function Rendering, specifically the implementation in the Mitsuba 3 renderer [S1, unofficial]. It builds on the differential path integral estimator framework, incorporating reparameterization techniques to handle the non-differentiable nature of ray-SDF intersections by warping the random samples. The "Path Replay Backpropagation" loop is a key innovation to efficiently compute gradients through the iterative sphere tracing process.

## Sources
[S1] Differentiable Signed Distance Function Rendering, https://github.com/rgl-epfl/differentiable-sdf-rendering, fetched 2026-10-26 (unofficial)
