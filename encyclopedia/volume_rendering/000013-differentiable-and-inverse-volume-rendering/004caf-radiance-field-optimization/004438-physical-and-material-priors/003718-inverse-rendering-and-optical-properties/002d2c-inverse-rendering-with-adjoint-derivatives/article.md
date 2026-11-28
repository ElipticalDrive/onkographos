# Inverse Rendering with Adjoint Derivatives

Also called: Adjoint-based Inverse Rendering

## Overview
Inverse rendering with adjoint derivatives is an optimization method that estimates scene optical properties (position, scale, rotation, cross-section, spherical harmonic coefficients) from images by computing gradients of the forward rendering pass via reverse-mode automatic differentiation [S1, unofficial]. It distinguishes itself from detached gradient methods by propagating gradients through refractive interfaces and complex scattering events, enabling the optimization of geometric parameters like lens heightfields where detached formulations are insufficient [S2, unofficial]. It builds on [Inverse Rendering and Optical Properties](7914-inverse-rendering-and-optical-properties.md).

## How it works
**Inputs:**
*   Target image(s).
*   Initial parameter set for optical properties (position, scale, rotation, cross-section, spherical harmonic coefficients) [S1, unofficial].
*   Scene geometry and camera parameters.

**Outputs:**
*   Optimized optical property parameters.

**Data Structures:**
*   **Primitives:** Represented as mathematically defined shapes (e.g., fuzzy spheres or ellipsoids) allowing closed-form calculation of ray traversal segments and transmittance [S1, unofficial].
*   **Ray Segments:** Light rays are split into segments within each primitive to compute scattering and absorption events deterministically [S1, unofficial].
*   **Heightfield:** For geometric inverse problems, the lens surface is represented as a heightfield grid [S2, unofficial].

**Steps:**
1.  **Initialization:** Start optimization with a small number of primitives. Gradually generate new primitives to add detail during the optimization process [S1, unofficial].
2.  **Forward Pass:** Trace rays through the scene. For each ray-primitive intersection, calculate the segment length and transmittance using closed-form solutions to avoid stochastic noise [S1, unofficial].
3.  **Gradient Computation:** Compute the gradient of the loss function with respect to the parameters using adjoint derivatives. This requires computing forward ray Jacobians during both the primal and backward passes [S2, unofficial].
4.  **Parameter Update:** Update the parameters using an optimizer.
    *   For general optical properties: Use Bound Adam, an improved Adam optimizer that respects parameter boundaries [S1, unofficial].
    *   For geometric lens optimization: Use standard Adam [Kingma and Ba 2014] [S2, unofficial].
5.  **Loss Calculation:** The loss function consists of L1 error and D-SSIM between the rendered image and the target image, plus regularization terms for anisotropy and density of small primitives [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_inverse_rendering(params, target_image, scene, num_iterations=1000):
    # Initialize optimizer with bounds if applicable
    optimizer = BoundAdam(params) if params.has_bounds else Adam(params)
    
    for i in range(num_iterations):
        # 1. Forward Rendering
        rendered_image = forward_render(scene, params)
        
        # 2. Compute Loss
        # L1 error and D-SSIM
        l1_loss = L1(rendered_image, target_image)
        dssim_loss = D_SSIM(rendered_image, target_image)
        
        # Regularization for anisotropy and density of small primitives
        reg_loss = regularization(params)
        
        total_loss = l1_loss + dssim_loss + reg_loss
        
        # 3. Backward Pass (Adjoint Derivatives)
        # Gradients propagate through refractive interfaces and scattering events
        # Requires forward ray Jacobians
        total_loss.backward()
        
        # 4. Update Parameters
        optimizer.step()
        optimizer.zero_grad()
        
        # Optional: Add primitives if using progressive initialization
        if i % add_primitives_interval == 0:
            params = add_primitives(params, scene)
            
    return params
```

**Parameters:**
*   **Optimizer:** Bound Adam (for bounded optical properties) or Adam [S1, unofficial][S2, unofficial].
*   **Iterations:** 1000 iterations for lens optimization [S2, unofficial].
*   **Path Length:** Fixed path length 4 for lens optimization [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Caustic Paths:** Caustic paths have very few valid sensor connections per path, which can affect gradient variance in reservoir-based methods [S2, unofficial].
*   **Memory Usage:** Naive AD and buffered two-pass variants grow linearly with path length and may exceed memory limits (e.g., 16 GB) for long paths [S2, unofficial].
*   **Noise:** Using closed-form solutions for transmittance eliminates noise associated with stochastic methods [S1, unofficial].

## Subtopics

### Geometric Inverse Problem (Lens Optimization)
Optimizes the heightfield of a dielectric lens to produce a target caustic pattern on a diffuse receiving plane [S2, unofficial]. It requires adjoint derivatives to propagate gradients through the refractive interface, as detached formulations cannot determine how shifting lens vertices alters downstream splat positions on the sensor [S2, unofficial]. It uses an Adam optimizer for 1000 iterations [S2, unofficial]. Among adjoint methods, `lrb_3pass` is fastest, followed by `reslrb` and `lrb_2pass`, while `ptracer` (naive AD) is fastest overall for fixed short path lengths [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Naive AD (`ptracer`) is fastest for short fixed path lengths (e.g., 4) due to no replay overhead [S2, unofficial]. Among adjoint methods, `lrb_3pass` is fastest, followed by `reslrb` and `lrb_2pass` [S2, unofficial]. The gap widens at greater path lengths due to buffer overhead and additional traversals [S2, unofficial].
*   **Memory Complexity:** Naive AD and buffered two-pass (`lrb_2pass`) grow linearly with path length and scene complexity, exceeding 16 GB limits for path lengths >32 and >128 respectively [S2, unofficial]. Constant-memory methods (`reslrb`, `lrb_3pass`) maintain a flat memory profile [S2, unofficial].
*   **Performance:** Closed-form transmittance calculation results in zero noise and faster rendering compared to stochastic methods [S1, unofficial]. Memory usage is tens to hundreds of times lower than voxel grids [S1, unofficial].
*   **Strengths:** Noiseless rendering, high speed, low memory (with constant-memory methods), ability to optimize geometric parameters through refractive interfaces.
*   **Limitations:** Linear memory growth for naive/buffered methods, complexity of implementing adjoint derivatives for complex scattering.

## Background
The method originates from research on differentiable light tracing and inverse rendering. Key developments include the use of volumetric ray-traced primitives for modeling scattering media [S1, unofficial] and constant-memory differentiable light tracing techniques [S2, unofficial]. It is used in applications such as radiance field optimization and complex camera simulation [S1, unofficial].

## Sources
[S1] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
[S2] Constant-Memory Differentiable Light Tracing, https://arxiv.org/abs/2609.32920, fetched 2026-10-29 (unofficial)
