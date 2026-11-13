# Dual-Pass Differentiable Volume Rendering with Sparse Gradient Recording

Also called: Analytical Differentiable Volume Rendering, Concurrent Forward/Backward Pass, Differentiable Renderer based on Differential RTE, Sparse gradRec Gradient Storage

## Overview
This method is a differentiable rendering technique that executes forward and backward passes concurrently to reuse the single-scattering Jacobian term $J_{ss}$ during backpropagation, utilizing a sparse structure `gradRec` to store gradients only along sampled ray paths rather than the entire volume grid [S1][S2, unofficial]. It sets itself apart from standard reverse-mode automatic differentiation by deriving an analytical differential expression for the radiance transport equation (RTE) to compute density derivatives directly, thereby avoiding the exclusive reliance on automatic differentiation and reducing memory usage from $O(N^3)$ to $O(RN)$ [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A volumetric density field (voxel grid), camera parameters (ray origins and directions), and lighting configuration (environment map or point/parallel lights) [G].
*   **Outputs:** Rendered image radiance $L_{out}$ and gradients with respect to the density field $\dot{\kappa}_t$ (or $\dot{\sigma}_t$) for optimization [S2, unofficial].

**Data Structures:**
*   **`gradRec`:** A sparse stack structure that stores gradients only along active sampled ray paths, not the full volume grid [S1][S2, unofficial]. The structure supports `push` and `pop` operations [G].
*   **Voxel Grid:** Uniform 3D grid where $v_p$ denotes a voxel position [S2, unofficial].

**Numbered Steps:**
1.  **Initialize:** For each pixel/ray $\mathbf{r}$, initialize transmittance $T=1$ and accumulated radiance $L=0$. Initialize the `gradRec` stack for the current ray [G].
2.  **Ray Marching (Concurrent Forward/Backward):** Iterate through samples $p$ along the ray path from near to far [G].
    *   **Forward Pass:** Calculate the single-scattering radiance contribution $L_{ss}$ and transmittance $\tau$ [S2, unofficial].
        *   Calculate $J_{ss}$ (the Jacobian of the single-scattering term with respect to density).
            *   If the scene uses an **environment map**: Estimate $J_{ss}$ using importance sampling over the unit sphere $S^2$ [S1][S2, unofficial].
            *   If the scene uses **point or parallel lights**: Calculate $J_{ss}$ directly without Monte Carlo integration, as lighting directions and radiance are known [S1][S2, unofficial].
        *   Accumulate radiance: $L_{out} \leftarrow L_{out} + \text{contribution}$ [S1][S2, unofficial].
    *   **Backward Pass / Gradient Recording:**
        *   Compute the derivative of transmittance $\dot{\tau}$ with respect to density. The formula is:
            $$
            \dot{\tau}(v_{x0}, v_{x1}) = -\tau(v_{x0}, v_{x1}) \sum_{p=x0}^{x1} \dot{\kappa}_t(v_p) \Delta d
            $$
            where $\dot{\kappa}_t = \sigma_t$ (extinction cross-section), $v_p$ is a voxel position, and $\Delta d$ is the sampling step length [S2, unofficial].
        *   Compute the derivative of single-scattering radiance $\dot{L}_{ss}$. Under the single-scattering assumption:
            $$
            \dot{L}_{ss}(x, \omega) = \sum_{p=x_a}^{x_b} \left( \dot{\tau}(v_p, v_{x_b}) \kappa_t(v_p) + \tau(v_p, v_{x_b}) \dot{\kappa}_t(v_p) \right) J_{ss}(v_p, \omega) \Delta d + \sum_{p=x_a}^{x_b} \tau(v_p, v_{x_b}) \kappa_t(v_p) \dot{J}_{ss}(v_p, \omega) \Delta d
            $$
            [S2, unofficial].
        *   **Push to Sparse Stack:** As intersections occur along the ray, push the computed gradients associated with the sampling positions onto the `gradRec` stack [S1][S2, unofficial]. This ensures gradients are stored only for active ray paths [S1][S2, unofficial].
3.  **Interface Derivative Handling:** When applying the Reynolds transport theorem to compute derivatives of integrals, disregard the derivative of the interface. This is valid because the work concerns purely participating media volumes where the incident radiance $L_{ri}$ exhibits continuity in $S^2$ [S2, unofficial].
4.  **Density Reconstruction (Post-Processing):**
    *   After the forward/backward pass completes for all rays, compute the loss function between the target and primal images [S1][S2, unofficial].
    *   Pop the `gradRec` stack to release memory [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dual_pass_differentiable_rendering(volume_grid, rays, lights):
    """
    volume_grid: 3D array of density/extinction coefficients sigma_t
    rays: list of ray objects, each with origin, direction, and samples
    lights: lighting configuration (environment_map or point_lights)
    """
    # Initialize output image
    rendered_image = [0.0] * len(rays)

    for ray_idx, ray in enumerate(rays):
        grad_rec_stack = []  # Sparse stack for this ray
        T = 1.0
        L_out = 0.0
        tau_dot_sum = 0.0  # Running sum for tau derivative

        for sample in ray.samples:
            pos = sample.position
            delta_d = sample.step_size
            sigma_t = volume_grid[pos]

            # --- Forward Pass ---
            # Calculate J_ss based on lighting type
            if lights.is_environment_map:
                # Standard importance sampling over unit sphere [G]
                J_ss = importance_sample_sphere_integration(lights.env_map, pos, ray.direction)
            else:
                # Direct calculation for point/parallel lights [G]
                J_ss = direct_lighting_calculation(lights.point_lights, pos, ray.direction)

            # Accumulate radiance contribution
            L_contribution = T * sigma_t * J_ss * delta_d
            L_out += L_contribution

            # Update transmittance for next step
            T *= math.exp(-sigma_t * delta_d)

            # --- Backward Pass / Gradient Recording ---
            # Compute derivative of transmittance segment [origin, current_pos]
            # tau_dot = -tau * sum(sigma_t * delta_d)
            tau_dot_sum += sigma_t * delta_d
            tau_dot = -T * tau_dot_sum

            # Compute gradient of L_out w.r.t sigma_t at this sample
            # From S2: dL/d_sigma involves tau_dot * kappa_t * J_ss and tau * kappa_dot * J_ss
            # Since kappa_dot = sigma_t, we compute the local gradient contribution
            grad_sigma = (tau_dot * sigma_t * J_ss + T * sigma_t * J_ss) * delta_d

            # Push to sparse stack as intersections occur
            grad_rec_stack.append((pos, grad_sigma))

        rendered_image[ray_idx] = L_out
        ray.grad_stack = grad_rec_stack

    # --- Density Reconstruction Step ---
    grad_accumulation_grid = [0.0] * len(volume_grid)

    for ray in rays:
        # Pop from stack to release memory
        while ray.grad_stack:
            pos, grad_val = ray.grad_stack.pop()
            grad_accumulation_grid[pos] += grad_val

    return rendered_image, grad_accumulation_grid

def importance_sample_sphere_integration(env_map, pos, direction):
    # Standard importance sampling over unit sphere [G]
    return env_map.sample(pos, direction)

def direct_lighting_calculation(point_lights, pos, direction):
    # Direct calculation for point/parallel lights [G]
    return sum(light.intensity * light.direction for light in point_lights)
```

**Parameters:**
*   **Sampling Step Size ($\Delta d$):** Not given in the sources. Typically determined by voxel size or adaptive sampling [G].
*   **Lighting Type:** Determines the method for calculating $J_{ss}$ (importance sampling vs. direct calculation) [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Monte Carlo Cost:** Straightforward Monte Carlo integration for sphere integration is computationally expensive. Mitigated by using importance sampling for environment maps and direct computation for point/parallel lights [S1][S2, unofficial].
*   **Interface Derivatives:** The derivative of the interface in the Reynolds transport theorem is disregarded. This is a valid assumption only for purely participating media volumes where incident radiance $L_{ri}$ exhibits continuity in $S^2$. If interfaces are present, this method may require modification [S2, unofficial].
*   **Memory Management:** The `gradRec` stack must be properly popped after the density field update to release memory. Failure to do so leads to memory leaks, although the complexity is still bounded by $O(RN)$ per frame if cleared [S1][S2, unofficial].

## Subtopics

### [Density Reconstruction via Differentiable Rendering](3659-density-reconstruction-via-differentiable-rendering.md)
Density Reconstruction via Differentiable Rendering is the optimization step that updates a volumetric density field $\rho$ by backpropagating image-space errors into the volume grid. It solves the inverse problem of recovering physical density from observed images by computing the gradient of a loss function with respect to voxel values and applying gradient descent. It sets itself apart from naive differentiable rendering by avoiding a separate backward pass or recomputation of the forward ray march; instead, it relies on gradient records stored during the forward pass (provided by [Dual-Pass Differentiable Volume Rendering with Sparse Gradient Recording](3558-dual-pass-differentiable-volume-rendering-with-sparse-gradie.md)) to scatter gradients directly to neighboring voxels.

## Operating characteristics
*   **Time Complexity:** The concurrent forward/backward pass allows for efficient reuse of $J_{ss}$. The complexity is dominated by ray marching and the calculation of $J_{ss}$. Importance sampling mitigates the computational expense of straightforward Monte Carlo integration for environment maps, while direct calculation is used for point/parallel lights [S1][S2, unofficial].
*   **Memory Complexity:** Reduced from $O(N^3)$ (storing gradients for all voxels) to $O(RN)$, where $N$ is the voxel grid dimension (or ray length) and $R$ is the number of rays. This is achieved by maintaining gradients only along active ray paths using the `gradRec` stack [S1][S2, unofficial].
*   **Strengths:** High computational efficacy for fluid density optimization; avoids the high memory cost of standard reverse-mode AD; analytical derivatives provide precise gradients without the noise of stochastic AD [S1][S2, unofficial].
*   **Limitations:** Assumes single-scattering or requires specific handling of multiple scattering (not detailed in the provided notes for the differential form); limited to purely participating media (no interface derivatives) [S2, unofficial].
*   **When to use:** When optimizing volumetric density fields from images (inverse rendering) and memory is a constraint, particularly for large voxel grids where $O(N^3)$ is prohibitive [S1][S2, unofficial].

## Background
*   **Origin:** The method is described in "Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video" [S1][S2, unofficial].
*   **History:** It builds upon the radiance transport equation (RTE) and its differential form. It incorporates the Reynolds transport theorem for computing derivatives of hydrodynamic integral equations [S2, unofficial].
*   **Usage:** Used for fluid density optimization and reconstruction from monocular video [S1][S2, unofficial].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
[S2] Physics-Based Differentiable Rendering for Efficient and ... - MDPI, https://www.mdpi.com/1099-4300/25/9/1348, fetched 2026-10-27 (unofficial)
