# Dual-Pass Differentiable Volume Rendering with Sparse Gradient Recording

Also called: Concurrent Forward/Backward Pass, Sparse gradRec Gradient Storage

## Overview
This method is a differentiable rendering technique that executes forward and backward passes concurrently to compute gradients for density field optimization [S1]. It solves the high memory cost of standard reverse-mode automatic differentiation by storing gradients only along active sampled ray paths in a sparse structure, rather than the entire volume grid [S1]. It sets itself apart from naive differentiable rendering by avoiding a separate backward pass or recomputation of the forward ray march; instead, it relies on gradient records stored during the forward pass to scatter gradients directly to neighboring voxels [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volumetric density field $\rho$, camera parameters, illumination sources (environment map or point/parallel lights), and a target image for loss computation.
*   **Outputs:** Rendered image, loss value, and gradients with respect to the density field $\frac{\partial L}{\partial \rho}$ [S1].

**Data Structures:**
*   **gradRec:** A sparse stack structure storing gradients only along active sampled ray paths [S1]. It does not store gradients for the full volume grid.
*   **J_ss:** The Jacobian term associated with sphere integration, reused during backpropagation [S1].

**Steps:**
1.  **Concurrent Pass Execution:** Execute the forward rendering pass and the backward gradient computation pass concurrently [S1]. This allows the reuse of the Jacobian term $J_{ss}$ during the backpropagation phase [S1].
2.  **Sphere Integration:** Perform integration over the unit sphere to calculate $J_{ss}$ [S1].
    *   For environment map illuminations, use importance sampling [S1].
    *   For point or parallel light illuminations, use direct computation [S1].
    *   *Note:* Straightforward Monte Carlo integration is computationally expensive and avoided [S1].
3.  **Ray Marching and Gradient Recording:** March rays through the volume to render the image [S1]. As intersections occur along the ray, push the associated gradients onto the `gradRec` stack [S1].
4.  **Density Reconstruction:** Compute the loss function between the rendered image and the target image [S1]. Use the gradients stored in `gradRec` to update the density field $\rho$ [S1].
5.  **Memory Release:** After the density field update finishes, pop the `gradRec` stack to release memory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dual_pass_render_and_optimize(volume_grid, rays, target_image, lights):
    # Initialize sparse gradient recorder
    gradRec = Stack()
    
    # Initialize density field gradients
    density_gradients = zeros_like(volume_grid)
    
    # Concurrent Forward and Backward Passes
    for ray in rays:
        # 1. Forward Pass: Render radiance
        radiance = 0.0
        transmittance = 1.0
        accumulated_gradients = []
        
        for sample in ray.march(volume_grid):
            density = sample.density
            color = sample.color
            step_size = sample.step_size
            
            # Standard volume rendering contribution
            alpha = 1.0 - exp(-density * step_size)
            radiance += transmittance * alpha * color
            transmittance *= (1.0 - alpha)
            
            # 2. Record gradients along the ray path
            # Compute local gradient contribution for this sample
            # Note: Exact derivative formula depends on the specific rendering equation
            # used in the parent method, but here we store the necessary terms
            local_grad = compute_gradient_contribution(sample, target_image)
            gradRec.push(local_grad)
            
        # 3. Backward Pass: Propagate gradients using stored records
        # The concurrent nature allows reuse of J_ss computed during forward pass
        # J_ss calculation depends on illumination type
        if ray.illumination_type == ENVIRONMENT_MAP:
            J_ss = importance_sampling_integration(ray, lights)
        else:
            J_ss = direct_computation_integration(ray, lights)
            
        # Scatter gradients from gradRec to volume grid
        while not gradRec.is_empty():
            grad_record = gradRec.pop()
            # Scatter gradient to the corresponding voxel(s)
            density_gradients.scatter(grad_record.position, grad_record.value)
            
    return density_gradients
```

**Parameters:**
*   **R:** Number of rays.
*   **N:** Length of each ray (number of samples per ray).
*   **Illumination Type:** Determines the integration method for $J_{ss}$ [S1].

**Edge Cases and Pitfalls:**
*   **Sphere Integration Cost:** Straightforward Monte Carlo integration for sphere integration is computationally expensive [S1]. Mitigated by using importance sampling for environment maps and direct computation for point/parallel lights [S1].
*   **Memory Management:** Failure to pop the `gradRec` stack after density field update leads to memory leaks [S1].

## Subtopics

### [Density Reconstruction via Differentiable Rendering](3659-density-reconstruction-via-differentiable-rendering.md)
Density Reconstruction via Differentiable Rendering is the optimization step that updates a volumetric density field $\rho$ by backpropagating image-space errors into the volume grid. It solves the inverse problem of recovering physical density from observed images by computing the gradient of a loss function with respect to voxel values and applying gradient descent. It sets itself apart from naive differentiable rendering by avoiding a separate backward pass or recomputation of the forward ray march; instead, it relies on gradient records stored during the forward pass (provided by [Dual-Pass Differentiable Volume Rendering with Sparse Gradient Recording](3558-dual-pass-differentiable-volume-rendering-with-sparse-gradie.md)) to scatter gradients directly to neighboring voxels.

## Operating characteristics
*   **Memory Complexity:** Reduces memory usage from $O(N^3)$ to $O(RN)$, where $N$ is the voxel grid dimension and $R$ is the number of rays [S1]. This is achieved by maintaining gradients only along active ray paths rather than the entire volume grid [S1].
*   **Time Complexity:** Concurrent execution of forward and backward passes allows reuse of intermediate terms like $J_{ss}$, potentially reducing redundant computations [S1].
*   **Strengths:** High computational efficacy for fluid density optimization; significant memory savings compared to storing full volume gradients [S1].
*   **Limitations:** Requires careful management of the `gradRec` stack; specialized integration techniques are needed for different illumination types [S1].

## Background
This method was proposed in the context of physics-based differentiable rendering for fluid modeling [S1]. It addresses the challenges of efficient and plausible fluid modeling from monocular video by optimizing the density field through differentiable rendering [S1].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
