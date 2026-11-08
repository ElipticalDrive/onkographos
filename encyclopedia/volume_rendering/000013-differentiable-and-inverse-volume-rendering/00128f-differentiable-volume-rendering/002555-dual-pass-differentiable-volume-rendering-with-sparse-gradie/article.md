# Dual-Pass Differentiable Volume Rendering with Sparse Gradient Recording

Also called: Analytical Differentiable Volume Rendering, Concurrent Forward/Backward Pass, Differentiable Renderer based on Differential RTE, Sparse gradRec Gradient Storage

## Overview
This method is a differentiable rendering technique that executes forward and backward passes concurrently to reuse the single-scattering Jacobian term $J_{ss}$ during backpropagation, utilizing a sparse structure `gradRec` to store gradients only along sampled ray paths rather than the entire volume grid [S1][S2, unofficial]. It sets itself apart from standard reverse-mode automatic differentiation by deriving an analytical differential expression for the radiance transport equation (RTE) to compute density derivatives directly, thereby avoiding the exclusive reliance on automatic differentiation and reducing memory usage from $O(N^3)$ to $O(RN)$ [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A volumetric density field (voxel grid), camera parameters (ray origins and directions), and lighting configuration (environment map or point/parallel lights).
*   **Outputs:** Rendered image radiance $L_{out}$ and gradients with respect to the density field $\dot{\kappa}_t$ (or $\dot{\sigma}_t$) for optimization.

**Data Structures:**
*   **`gradRec`:** A sparse stack structure. It stores gradients only along active sampled ray paths. It does not store gradients for the entire volume grid. The structure supports `push` and `pop` operations.
*   **Voxel Grid:** Uniform 3D grid where $v_p$ denotes a voxel position.

**Numbered Steps:**
1.  **Initialize:** For each pixel/ray $\mathbf{r}$, initialize transmittance $T=1$ and accumulated radiance $L=0$. Initialize the `gradRec` stack for the current ray.
2.  **Ray Marching (Concurrent Forward/Backward):** Iterate through samples $p$ along the ray path from near to far (or as defined by the marching order).
    *   **Forward Pass:** Calculate the single-scattering radiance contribution $L_{ss}$ and transmittance $\tau$.
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
        *   **Push to Sparse Stack:** As intersections occur along the ray, push the computed gradients associated with the sampling positions onto the `gradRec` stack [S1][S2, unofficial]. This ensures gradients are stored only for active ray paths.
3.  **Interface Derivative Handling:** When applying the Reynolds transport theorem to compute derivatives of integrals, disregard the derivative of the interface. This is valid because the work concerns purely participating media volumes where the incident radiance $L_{ri}$ exhibits continuity in $S^2$ [S2, unofficial].
4.  **Density Reconstruction (Post-Processing):**
    *   After the forward/backward pass completes for all rays, compute the loss function between the target and primal images.
    *   Pop the `gradRec` stack to release memory and scatter the accumulated gradients to the corresponding voxels in the density field for update [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dual_pass_differentiable_rendering(volume_grid, rays, lights):
    """
    volume_grid: 3D array of density/extinction coefficients sigma_t
    rays: list of ray objects, each with origin, direction, and samples
    lights: lighting configuration (environment_map or point_lights)
    """
    # Initialize output image
    rendered_image = zeros_like(rays.pixel_coords)
    
    # Note: gradRec is a stack per ray, managed during marching
    # In a real implementation, this might be a global sparse structure 
    # indexed by ray_id and sample_index, or a list of stacks.
    
    for ray in rays:
        grad_rec_stack = []  # Sparse stack for this ray
        T = 1.0
        L_out = 0.0
        
        # Ray marching loop
        for sample in ray.samples:
            pos = sample.position
            delta_d = sample.step_size
            sigma_t = volume_grid[pos]
            
            # --- Forward Pass ---
            # Calculate J_ss based on lighting type
            if lights.is_environment_map:
                J_ss = importance_sample_sphere_integration(lights.env_map, pos, ray.direction)
            else:
                J_ss = direct_lighting_calculation(lights.point_lights, pos, ray.direction)
            
            # Calculate transmittance derivative component (intermediate)
            # Note: Full tau derivative requires sum over previous steps, 
            # but we store necessary components for backward pass.
            # For simplicity in this concurrent step, we compute local contributions.
            
            # Accumulate Radiance (Standard DVR or Single Scattering approximation)
            # Using the differential form logic from S2 for L_ss
            # L_ss contribution = tau * kappa_t * J_ss * delta_d
            # We need tau from near to current point. 
            # Assuming T holds tau from origin to current pos.
            
            L_contribution = T * sigma_t * J_ss * delta_d
            L_out += L_contribution
            
            # Update Transmittance for next step
            T *= exp(-sigma_t * delta_d)
            
            # --- Backward Pass / Gradient Recording ---
            # Compute gradient of L_out w.r.t sigma_t at this sample
            # From S2: derivative involves tau_dot and tau * kappa_dot
            
            # 1. Compute tau_dot for the segment [origin, current_pos]
            # tau_dot(v_x0, v_x1) = -tau(v_x0, v_x1) * sum(dot_kappa_t * delta_d)
            # Since we are marching, we can maintain a running sum of dot_kappa_t * delta_d
            # However, S2 defines tau_dot for a segment. 
            # Let's assume we have access to the transmittance from current pos to end (T_end)
            # or we compute gradients backwards. 
            # The method states "concurrent", so we push gradients as we go.
            
            # Gradient of L_out w.r.t sigma_t at pos:
            # dL/d_sigma = dL/d_tau * d_tau/d_sigma + dL/d_Jss * d_Jss/d_sigma (if Jss depends on sigma)
            # Usually Jss is lighting dependent, not density dependent directly, 
            # but tau depends on sigma.
            
            # From S2 Eq 9, the term involving kappa_dot (sigma_t) is:
            # tau(v_p, v_xb) * kappa_dot_t(v_p) * J_ss
            # And the term involving tau_dot is:
            # tau_dot(v_p, v_xb) * kappa_t(v_p) * J_ss
            
            # To implement "concurrent" push, we need the gradient value.
            # Let's assume we have the upstream gradient dL_out/dL_current.
            # For a simple single-scattering model:
            grad_sigma = T * J_ss * delta_d  # Partial derivative of L_contribution w.r.t sigma_t
            
            # Push to sparse stack
            grad_rec_stack.append((pos, grad_sigma))
            
        # End Ray Marching
        
        # Store final radiance
        rendered_image[ray.pixel_index] = L_out
        
        # Store the stack for this ray to be processed later
        ray.grad_stack = grad_rec_stack

    # --- Density Reconstruction Step ---
    # After all rays are processed, update the volume grid
    # Initialize gradient accumulation grid
    grad_accumulation_grid = zeros_like(volume_grid)
    
    for ray in rays:
        # Pop from stack and scatter gradients
        # S1/S2: "When a density field update finishes, the stack is popped"
        while ray.grad_stack:
            pos, grad_val = ray.grad_stack.pop()
            grad_accumulation_grid[pos] += grad_val
            
    # Apply gradients to volume grid (e.g., Gradient Descent)
    # volume_grid -= learning_rate * grad_accumulation_grid
    
    return rendered_image, grad_accumulation_grid
```

**Parameters:**
*   **Sampling Step Size ($\Delta d$):** Not given in the sources. Typically determined by voxel size or adaptive sampling.
*   **Lighting Type:** Determines the method for calculating $J_{ss}$ (importance sampling vs. direct calculation) [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Monte Carlo Cost:** Straightforward Monte Carlo integration for sphere integration is computationally expensive. Mitigated by using importance sampling for environment maps and direct computation for point/parallel lights [S1][S2, unofficial].
*   **Interface Derivatives:** The derivative of the interface in the Reynolds transport theorem is disregarded. This is a valid assumption only for purely participating media volumes where incident radiance $L_{ri}$ exhibits continuity in $S^2$. If interfaces are present, this method may require modification [S2, unofficial].
*   **Memory Management:** The `gradRec` stack must be properly popped after the density field update to release memory. Failure to do so leads to memory leaks, although the complexity is still bounded by $O(RN)$ per frame if cleared [S1][S2, unofficial].

## Subtopics

### [Density Reconstruction via Differentiable Rendering](3659-density-reconstruction-via-differentiable-rendering.md)
Density Reconstruction via Differentiable Rendering is the optimization step that updates a volumetric density field $\rho$ by backpropagating image-space errors into the volume grid. It solves the inverse problem of recovering physical density from observed images by computing the gradient of a loss function with respect to voxel values and applying gradient descent. It sets itself apart from naive differentiable rendering by avoiding a separate backward pass or recomputation of the forward ray march; instead, it relies on gradient records stored during the forward pass (provided by [Dual-Pass Differentiable Volume Rendering with Sparse Gradient Recording](3558-dual-pass-differentiable-volume-rendering-with-sparse-gradie.md)) to scatter gradients directly to neighboring voxels.

## Operating characteristics
*   **Time Complexity:** The concurrent forward/backward pass allows for efficient reuse of $J_{ss}$. The complexity is dominated by ray marching and the calculation of $J_{ss}$. Importance sampling for environment maps adds overhead compared to direct lighting [S1][S2, unofficial].
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
