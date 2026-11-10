# Sensitivity AOV

Also called: Adjoint Pass, Sensitivity as an Arbitrary Output Variable, Sensitivity AOVs via Reverse-Mode Differentiation, Sensitivity Buffer

## Overview
Sensitivity AOV is a differentiable rendering technique that outputs scene parameter sensitivities as arbitrary output variables (AOVs) by computing a single adjoint pass per objective, which is then re-projected for multiple inspection viewpoints [S1, unofficial]. It solves the problem of human interpretability in inverse rendering by exposing which parts of a scene matter for a given goal, analogous to how primal AOVs expose intermediate light transport quantities [S1, unofficial]. Unlike forward-mode derivatives which answer "what a single parameter does to the image," sensitivity AOVs answer "which parameters matter for this objective" by being dense in parameters but sparse in objectives, and unlike transport views, placement views can re-project a finished buffer rather than accumulating contributions during the adjoint sweep [S1, unofficial].

## How it works
The method separates the fixed camera that defines the scalar objective from the free camera used to inspect the result. It relies on a reverse-mode differentiation pass to populate a sensitivity buffer, from which various views (object-level, parameter-type, per-texel) are derived without recomputing gradients [S1, unofficial].

### Inputs and Outputs
**Inputs:**
1.  **Scene Parameters ($\theta$):** A hierarchical set of parameters (e.g., `floor.bsdf.reflectance`). Leaves are individual parameters; interior nodes are materials or objects [S1, unofficial].
2.  **Objective Camera:** The fixed camera defining the scalar objective $M$. It determines the primal image $I$ and the adjoint seed $\frac{\partial M}{\partial I}$ [S1, unofficial].
3.  **Inspection Camera:** A free camera used only for viewing the sensitivity AOVs. Moving it does not change the objective or require re-differentiation [S1, unofficial].
4.  **Scalar Objective ($M$):** A function evaluated on the primal image $I$ [S1, unofficial].

**Outputs:**
1.  **Primal Image ($I$):** $I = R(\theta)$, where $R$ is the renderer [S1, unofficial].
2.  **Sensitivity Buffer:** A tree structure mirroring the scene's parameter hierarchy, storing the sensitivity $g = \frac{\partial M}{\partial \theta}$ for every parameter component [S1, unofficial].
3.  **Sensitivity AOVs:** Render outputs derived from the buffer via selection, aggregation, and binding [S1, unofficial].

### Data Structures
**Sensitivity Buffer:**
A tree structure where:
*   **Leaves:** Individual scene parameters (e.g., a single scalar reflectance value or a specific texel in a texture).
*   **Interior Nodes:** Materials or objects grouping their parameters.
*   **Values:** Scalar sensitivities $\frac{\partial M}{\partial \theta_i}$ for each leaf [S1, unofficial].

**Primary Visibility Data:**
For binding uniform parameters, a one-sample-per-pixel primary-visibility pass yields:
*   **Object Identity:** The ID of the object hit by the ray.
*   **UV Coordinates:** Texture coordinates for spatially varying parameters [S1, unofficial].

### Numbered Steps

1.  **Evaluate Primal Image:** Render the scene using the objective camera to obtain the primal image $I = R(\theta)$ [S1, unofficial].
2.  **Evaluate Objective:** Compute the scalar objective value $M(I)$ [S1, unofficial].
3.  **Compute Adjoint Seed:** Calculate the gradient of the objective with respect to the image, $\frac{\partial M}{\partial I}$ [S1, unofficial].
4.  **Reverse-Mode Pass (Adjoint Sweep):** Perform a single reverse-mode differentiation pass propagating $\frac{\partial M}{\partial I}$ back through the light transport simulation. This yields the sensitivity vector $g$ for every scene parameter $\theta$ [S1, unofficial]:
    $$
    g = \frac{\partial M(R(\theta))}{\partial \theta} = \frac{\partial M}{\partial I} \frac{\partial I}{\partial \theta}
    $$
    This step populates the **Sensitivity Buffer** [S1, unofficial]. The cost is largely independent of the number of parameters [S1, unofficial].
5.  **Primary Visibility Pass (for Placement Views):** Trace one ray per pixel from the inspection camera to identify the hit object and retrieve UV coordinates. This pass carries no gradient computation [S1, unofficial].
6.  **Selection and Aggregation:** Query the sensitivity buffer to select a subtree (e.g., all parameters of one object, or all reflectance parameters across the scene). Aggregate the selected sensitivities into a single scalar per group. The standard aggregation uses the $\ell_2$ norm of the value-weighted gradients:
    $$
    \text{norm} = \left\| \theta \frac{\partial M}{\partial \theta} \right\|_2
    $$
    This measures the total response to perturbing the whole group proportionally [S1, unofficial]. For spatially varying parameters, per-texel gradients are carried to the surface via UVs. To suppress fireflies and fill gaps caused by independent sampling of gradients and visible texels, the per-texel field is pooled before projection [S1, unofficial].
7.  **Binding:**
    *   **Uniform Parameters:** Bind the aggregated scalar to pixels using the object identity from the primary visibility pass. Paint the sensitivity of the hit object [S1, unofficial].
    *   **Spatially Varying Parameters:** Bind per-texel gradients to the surface using the hit UV coordinates [S1, unofficial].
8.  **Encoding:** Apply tone mapping and normalization. A critical choice is whether to use raw gradients $\frac{\partial M}{\partial \theta}$ or value-weighted gradients $\theta \frac{\partial M}{\partial \theta}$. The encoding must be recorded with the channel [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sensitivity_aov(scene, objective_camera, inspection_camera, objective_func):
    """
    Computes the sensitivity buffer and generates placement views.
    
    Args:
        scene: Scene object containing hierarchical parameters theta.
        objective_camera: Camera defining the objective M.
        inspection_camera: Camera for viewing the sensitivity AOV.
        objective_func: Function M(I) evaluating the scalar objective on image I.
        
    Returns:
        sensitivity_buffer: Tree of sensitivities.
        sensitivity_aov: Image of sensitivities bound to the inspection view.
    """
    
    # Step 1: Evaluate Primal Image
    # R is the differentiable renderer
    I = render(scene, objective_camera)
    
    # Step 2: Evaluate Objective
    M_val = objective_func(I)
    
    # Step 3: Compute Adjoint Seed
    # dM_dI is the gradient of the objective w.r.t. the image pixels
    dM_dI = compute_gradient(objective_func, I)
    
    # Step 4: Reverse-Mode Pass
    # Propagate dM_dI back through light transport to get sensitivity for all theta
    # This populates the sensitivity buffer
    sensitivity_buffer = reverse_mode_differentiation(scene, I, dM_dI)
    
    # Step 5: Primary Visibility Pass (for Placement Views)
    # One sample per pixel to get object IDs and UVs
    visibility_data = trace_primary_visibility(scene, inspection_camera, samples_per_pixel=1)
    
    # Step 6 & 7: Selection, Aggregation, and Binding
    # Example: Per-Object Sensitivity AOV
    # Select subtree for each object, aggregate, and bind
    
    # Initialize output image
    aov_image = zeros_like(I)
    
    for pixel in pixels(inspection_camera):
        ray = get_ray(inspection_camera, pixel)
        hit = trace_ray(scene, ray)
        
        if hit is not None:
            obj_id = hit.object_id
            uv = hit.uv
            
            # Select parameters for this object from the buffer
            # This is a vertical cut in the hierarchy tree
            obj_params = sensitivity_buffer.select_subtree(obj_id)
            
            # Aggregate: L2 norm of value-weighted gradients
            # theta_i is the value of the parameter, g_i is the sensitivity
            # norm = || theta * g ||_2
            weighted_grads = []
            for param in obj_params:
                theta_val = param.value
                g_val = param.sensitivity
                weighted_grads.append(theta_val * g_val)
            
            if weighted_grads:
                aggregated_sensitivity = l2_norm(weighted_grads)
            else:
                aggregated_sensitivity = 0.0
            
            # Encode and paint
            # Note: Encoding choice (raw vs weighted) must be consistent
            aov_image[pixel] = encode_sensitivity(aggregated_sensitivity)
            
        else:
            aov_image[pixel] = 0.0

    return sensitivity_buffer, aov_image

def trace_primary_visibility(scene, camera, samples_per_pixel):
    """
    Traces primary rays to get object IDs and UVs.
    Returns a structure mapping pixel coordinates to (object_id, uv).
    """
    visibility_data = {}
    for pixel in pixels(camera):
        # Trace one ray per pixel
        ray = get_ray(camera, pixel)
        hit = trace_ray(scene, ray)
        if hit:
            visibility_data[pixel] = (hit.object_id, hit.uv)
        else:
            visibility_data[pixel] = (None, None)
    return visibility_data

def l2_norm(values):
    """Computes the L2 norm of a list of values."""
    sum_sq = sum(v**2 for v in values)
    return sqrt(sum_sq)

def encode_sensitivity(value):
    """
    Encodes sensitivity for display.
    Placeholder for specific tone mapping/normalization.
    """
    return value
```

### Parameters
*   **Primary-visibility pass sample count:** One sample per pixel [S1, unofficial].
*   **Objective Camera:** Fixed camera defining $M$ [S1, unofficial].
*   **Inspection Camera:** Free camera for viewing [S1, unofficial].
*   **Encoding Weight:** Parameters can be weighted by their value ($\theta \frac{\partial M}{\partial \theta}$) to report response to proportional perturbation [S1, unofficial].

### Edge Cases and Pitfalls
*   **Visibility Discontinuities:** Placement views read interior gradients and do not treat visibility discontinuities, making boundary-correct geometric sensitivity out of scope [S1, unofficial].
*   **Parameter Interactions:** Sensitivities are local first-order quantities and do not capture interactions between parameters [S1, unofficial].
*   **Monte Carlo Variance:** Derivatives inherit path-tracing variance. Buffers at low sample counts may need denoising, but denoising for faithful inspection without reordering dominant parameters is an unstudied problem [S1, unofficial].
*   **Comparability:** Two sensitivity maps are only comparable if they share the same encoding (e.g., both weighted by parameter value). Comparing sensitivities across different parameter types (e.g., scalar reflectance vs. million-texel texture) requires careful normalization because their scales differ [S1, unofficial].
*   **Storage:** Multiple objectives require separate adjoint propagation and buffers unless the renderer carries several adjoint channels through one sweep; storage grows linearly with the number of objectives [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The sensitivity buffer is computed at the cost of one backward pass, largely independent of the number of scene parameters [S1, unofficial]. Generating placement views requires only a primary visibility pass, which is cheap compared to the adjoint sweep [S1, unofficial].
*   **Memory Complexity:** Storage grows linearly with the number of scene parameters (for the buffer) and linearly with the number of objectives if multiple buffers are kept [S1, unofficial].
*   **Strengths:**
    *   Allows human inspection of which parts of a scene matter for a given goal [S1, unofficial].
    *   One adjoint pass supports many inspection viewpoints (deferred shading analogy) [S1, unofficial].
    *   Dense in parameters, sparse in objectives [S1, unofficial].
*   **Limitations:**
    *   Does not capture parameter interactions [S1, unofficial].
    *   Placement views do not handle visibility discontinuities [S1, unofficial].
    *   Transport views (painting where influence lands) are not supported in the current implementation and require accumulation during the adjoint sweep [S1, unofficial].
    *   Denoising for human inspection is an open problem [S1, unofficial].
*   **When to use:** When you need to understand which scene parameters drive a specific objective for a fixed camera, and you want to inspect these sensitivities from multiple viewpoints without re-differentiating [S1, unofficial].
*   **When not to use:** When you need to see where a parameter's influence lands (transport view), or when you need to analyze interactions between parameters [S1, unofficial].

## Background
Sensitivity AOV was introduced by Linas Beresna and Eugene Fiume in 2026 [S1, unofficial]. It builds on the concept of Arbitrary Output Variables (AOVs) used in rendering to decompose and inspect intermediate quantities [S1, unofficial]. The method is implemented on a physically based path tracer using Dr.Jit and Path Replay Backpropagation [S1, unofficial]. It positions reverse-mode attribution against its forward-mode dual, arguing that sensitivity AOVs deserve first-class output status alongside the primal image [S1, unofficial].

## Sources
[S1] Sensitivity as an Arbitrary Output Variable for Differentiable Rendering, https://doi.org/10.1145/3829339.3847832, fetched 2026-10-30 (unofficial)
