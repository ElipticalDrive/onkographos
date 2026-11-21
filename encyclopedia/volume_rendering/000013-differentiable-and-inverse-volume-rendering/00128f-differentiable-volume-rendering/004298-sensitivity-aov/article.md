# Sensitivity AOV

Also called: Adjoint Pass, Sensitivity as an Arbitrary Output Variable, Sensitivity AOVs via Reverse-Mode Differentiation, Sensitivity Buffer

## Overview
Sensitivity AOV is a differentiable rendering technique that outputs scene parameter sensitivities as arbitrary output variables (AOVs) by computing a single adjoint pass per objective, which is then re-projected for multiple inspection viewpoints [S1, unofficial]. It solves the problem of human interpretability in inverse rendering by exposing which parts of a scene matter for a given goal, analogous to how primal AOVs expose intermediate light transport quantities [S1, unofficial]. Unlike forward-mode derivatives which answer "what a single parameter does to the image," sensitivity AOVs answer "which parameters matter for this objective" by being dense in parameters but sparse in objectives, and unlike transport views, placement views can re-project a finished buffer rather than accumulating contributions during the adjoint sweep [S1, unofficial]. The method separates the fixed camera that defines the scalar objective from the free camera used to inspect the result [S1, unofficial].

## How it works
The method relies on a reverse-mode differentiation pass to populate a sensitivity buffer, from which various views (object-level, parameter-type, per-texel) are derived without recomputing gradients [S1, unofficial].

### Inputs and Outputs
**Inputs:**
1. **Scene Parameters ($\theta$):** A hierarchical set of parameters (e.g., `floor.bsdf.reflectance`) where leaves are individual parameters and interior nodes are materials or objects [S1, unofficial].
2. **Objective Camera:** The fixed camera defining the scalar objective $M$ [S1, unofficial].
3. **Inspection Camera:** A free camera used only for viewing the sensitivity AOVs [S1, unofficial].
4. **Scalar Objective ($M$):** A function evaluated on the primal image $I$ [S1, unofficial].

**Outputs:**
1. **Primal Image ($I$):** $I = R(\theta)$, where $R$ is the renderer [S1, unofficial].
2. **Sensitivity Buffer:** A tree structure mirroring the scene's parameter hierarchy, storing the sensitivity $g = \frac{\partial M}{\partial \theta}$ for every parameter component [S1, unofficial].
3. **Sensitivity AOVs:** Render outputs derived from the buffer via selection, aggregation, and binding [S1, unofficial].

### Data Structures
**Sensitivity Buffer:**
A tree structure where:
* **Leaves:** Individual scene parameters (e.g., a single scalar reflectance value or a specific texel in a texture).
* **Interior Nodes:** Materials or objects grouping their parameters [S1, unofficial].
* **Values:** Scalar sensitivities $\frac{\partial M}{\partial \theta_i}$ for each leaf [S1, unofficial].

**Primary Visibility Data:**
For binding uniform parameters, a one-sample-per-pixel primary-visibility pass yields:
* **Object Identity:** The ID of the object hit by the ray [S1, unofficial].
* **UV Coordinates:** Texture coordinates for spatially varying parameters [S1, unofficial].

### Numbered Steps
1. **Evaluate Primal Image:** Render the scene using the objective camera to obtain the primal image $I = R(\theta)$ [S1, unofficial].
2. **Evaluate Objective:** Compute the scalar objective value $M(I)$ [S1, unofficial].
3. **Compute Adjoint Seed:** Calculate the gradient of the objective with respect to the image, $\frac{\partial M}{\partial I}$ [S1, unofficial].
4. **Reverse-Mode Pass (Adjoint Sweep):** Perform a single reverse-mode differentiation pass propagating $\frac{\partial M}{\partial I}$ back through the light transport simulation [S1, unofficial]. This yields the sensitivity vector $g$ for every scene parameter $\theta$ [S1, unofficial]:
   $$
   g = \frac{\partial M(R(\theta))}{\partial \theta} = \frac{\partial M}{\partial I} \frac{\partial I}{\partial \theta}
   $$
   This step populates the **Sensitivity Buffer** [S1, unofficial]. The cost is largely independent of the number of parameters [S1, unofficial].
5. **Primary Visibility Pass (for Placement Views):** Trace one ray per pixel from the inspection camera to identify the hit object and retrieve UV coordinates [S1, unofficial]. This pass carries no gradient computation [S1, unofficial].
6. **Selection and Aggregation:** Query the sensitivity buffer to select a subtree (e.g., all parameters of one object, or all reflectance parameters across the scene) [S1, unofficial]. Aggregate the selected sensitivities into a single scalar per group [S1, unofficial]. The standard aggregation uses the $\ell_2$ norm of the value-weighted gradients:
   $$
   \text{norm} = \left\| \theta \frac{\partial M}{\partial \theta} \right\|_2
   $$
   This measures the total response to perturbing the whole group proportionally [S1, unofficial]. For spatially varying parameters, per-texel gradients are carried to the surface via UVs [S1, unofficial]. To suppress fireflies and fill gaps caused by independent sampling of gradients and visible texels, the per-texel field is pooled before projection [S1, unofficial].
7. **Binding:**
   * **Uniform Parameters:** Bind the aggregated scalar to pixels using the object identity from the primary visibility pass [S1, unofficial]. Paint the sensitivity of the hit object [S1, unofficial].
   * **Spatially Varying Parameters:** Bind per-texel gradients to the surface using the hit UV coordinates [S1, unofficial].
8. **Encoding:** Apply tone mapping and normalization [S1, unofficial]. A critical choice is whether to use raw gradients $\frac{\partial M}{\partial \theta}$ or value-weighted gradients $\theta \frac{\partial M}{\partial \theta}$ [S1, unofficial]. The encoding must be recorded with the channel [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_sensitivity_aov(scene, objective_camera, inspection_camera, objective_func):
    # Step 1: Evaluate Primal Image
    I = path_trace(scene, objective_camera)
    
    # Step 2: Evaluate Objective
    M_val = objective_func(I)
    
    # Step 3: Compute Adjoint Seed
    dM_dI = autograd_gradient(objective_func, I)
    
    # Step 4: Reverse-Mode Pass
    sensitivity_buffer = adjoint_sweep(scene, dM_dI)
    
    # Step 5: Primary Visibility Pass
    visibility_data = trace_primary_visibility(scene, inspection_camera)
    
    # Step 6 & 7: Selection, Aggregation, and Binding
    aov_image = Image(inspection_camera.height, inspection_camera.width, channels=1)
    
    for y in range(inspection_camera.height):
        for x in range(inspection_camera.width):
            ray = inspection_camera.ray_at(x, y)
            hit = scene.intersect(ray)
            
            if hit is not None:
                obj_id = hit.object_id
                uv = hit.uv
                
                obj_params = sensitivity_buffer.select_subtree(obj_id)
                
                weighted_grads = []
                for param in obj_params:
                    theta_val = param.value
                    g_val = param.sensitivity
                    weighted_grads.append(theta_val * g_val)
                
                if weighted_grads:
                    aggregated = math_l2_norm(weighted_grads)
                else:
                    aggregated = 0.0
                
                aov_image.set_pixel(x, y, encode_sensitivity(aggregated))
            else:
                aov_image.set_pixel(x, y, 0.0)

    return sensitivity_buffer, aov_image

def trace_primary_visibility(scene, camera):
    visibility_data = {}
    for y in range(camera.height):
        for x in range(camera.width):
            ray = camera.ray_at(x, y)
            hit = scene.intersect(ray)
            if hit:
                visibility_data[(x, y)] = (hit.object_id, hit.uv)
            else:
                visibility_data[(x, y)] = (None, None)
    return visibility_data

def math_l2_norm(values):
    return math.sqrt(sum(v**2 for v in values))

def encode_sensitivity(value):
    # Encoding choice (raw vs weighted) must be recorded with the channel [G]
    return value

def path_trace(scene, camera):
    # Standard differentiable path tracing to compute I = R(theta) [G]
    pass

def autograd_gradient(func, x):
    # Standard reverse-mode automatic differentiation to compute dM/dx [G]
    pass

def adjoint_sweep(scene, dM_dI):
    # Standard adjoint pass propagating dM/dI back to scene parameters [G]
    pass
```

### Parameters
* **Primary-visibility pass sample count:** One sample per pixel [S1, unofficial].
* **Objective Camera:** Fixed camera defining $M$ [S1, unofficial].
* **Inspection Camera:** Free camera for viewing [S1, unofficial].
* **Encoding Weight:** Parameters can be weighted by their value ($\theta \frac{\partial M}{\partial \theta}$) to report response to proportional perturbation [S1, unofficial].

### Edge Cases and Pitfalls
* **Visibility Discontinuities:** Placement views read interior gradients and do not treat visibility discontinuities, making boundary-correct geometric sensitivity out of scope [S1, unofficial].
* **Parameter Interactions:** Sensitivities are local first-order quantities and do not capture interactions between parameters [S1, unofficial].
* **Monte Carlo Variance:** Derivatives inherit path-tracing variance [S1, unofficial]. Buffers at low sample counts may need denoising, but denoising for faithful inspection without reordering dominant parameters is an unstudied problem [S1, unofficial].
* **Comparability:** Two sensitivity maps are only comparable if they share the same encoding (e.g., both weighted by parameter value) [S1, unofficial]. Comparing sensitivities across different parameter types (e.g., scalar reflectance vs. million-texel texture) requires careful normalization because their scales differ [S1, unofficial].
* **Storage:** Multiple objectives require separate adjoint propagation and buffers unless the renderer carries several adjoint channels through one sweep; storage grows linearly with the number of objectives [S1, unofficial].

## Operating characteristics
* **Time Complexity:** The sensitivity buffer is computed at the cost of one backward pass, largely independent of the number of scene parameters [S1, unofficial]. Generating placement views requires only a primary visibility pass, which is cheap compared to the adjoint sweep [S1, unofficial].
* **Memory Complexity:** Storage grows linearly with the number of scene parameters (for the buffer) and linearly with the number of objectives if multiple buffers are kept [S1, unofficial].
* **Strengths:**
  * Allows human inspection of which parts of a scene matter for a given goal [S1, unofficial].
  * One adjoint pass supports many inspection viewpoints (deferred shading analogy) [S1, unofficial].
  * Dense in parameters, sparse in objectives [S1, unofficial].
* **Limitations:**
  * Does not capture parameter interactions [S1, unofficial].
  * Placement views do not handle visibility discontinuities [S1, unofficial].
  * Transport views (painting where influence lands) are not supported in the current implementation and require accumulation during the adjoint sweep [S1, unofficial].
  * Denoising for human inspection is an open problem [S1, unofficial].
* **When to use:** When you need to understand which scene parameters drive a specific objective for a fixed camera, and you want to inspect these sensitivities from multiple viewpoints without re-differentiating [S1, unofficial].
* **When not to use:** When you need to see where a parameter's influence lands (transport view), or when you need to analyze interactions between parameters [S1, unofficial].

## Background
Sensitivity AOV was introduced by Linas Beresna and Eugene Fiume in 2026 [S1, unofficial]. It builds on the concept of Arbitrary Output Variables (AOVs) used in rendering to decompose and inspect intermediate quantities [S1, unofficial]. The method is implemented on a physically based path tracer using Dr.Jit and Path Replay Backpropagation [S1, unofficial]. It positions reverse-mode attribution against its forward-mode dual, arguing that sensitivity AOVs deserve first-class output status alongside the primal image [S1, unofficial].

## Sources
[S1] Sensitivity as an Arbitrary Output Variable for Differentiable Rendering, https://doi.org/10.1145/3829339.3847832, fetched 2026-10-30 (unofficial)
