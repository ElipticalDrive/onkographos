# Background Volume Rendering with Sparse Grid


## Overview
Background Volume Rendering with Sparse Grid is a technique for rendering unbounded or distant background elements (e.g., sky, distant geometry) using a separate sparse grid structure distinct from the foreground scene representation [S1, unofficial]. It allows for efficient rendering of background content by treating it as a separate volumetric field accessed via ray traversal, enabling different resolution and sampling strategies for background versus foreground [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins and directions, background sparse grid data (`grid.background_data`), background grid links (`grid.background_links`), optional gradient output (`grad_output`), and optimization parameters (`opt.stop_thresh`, `sparsity_loss`) [S1, unofficial].
*   **Outputs:** Rendered RGB color and confidence/confidence map for the background contribution; gradients for background density/color (`grad_background_out`) and mask/visibility (`mask_background_out`) if in backward mode [S1, unofficial].

**Data Structures:**
*   **PackedGridOutputGrads:** A structure containing:
    *   `grad_background_out`: Gradient for background density/color data.
    *   `mask_background_out`: Gradient for background mask/visibility data [S1, unofficial].
*   **Background Grid:** Accessed via `grid.background_links` and `grid.background_data`, distinct from foreground grids [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Initialization and Bounds:**
    For each ray, determine the spatial bounds of the intersection with the background grid using `ray_find_bounds_bg` [S1, unofficial].

2.  **Transmittance Check (Skip Condition):**
    If the logarithmic transmittance `log_transmit` is less than `-25.0`, skip the ray entirely to save computation, assuming negligible contribution [S1, unofficial].
    $$ \text{if } \log\_transmit < -25.0 \text{, return} $$

3.  **Forward Pass (`render_background_forward`):**
    Iterate through ray samples within the background bounds.
    *   **Accumulation:** Accumulate color and transmittance. The color contribution is computed as:
        $$ total\_color += \max(color, 0.0) \times weight $$
        Where `weight` is the transmission factor for the current step [S1, unofficial]. Note: The source snippet mentions `grad_output[i]` in the accumulation formula, but clarifies this likely refers to a weight/transmission factor in this specific context or is a variable name reuse; standard volume rendering uses `weight * color` [S1, unofficial].
    *   **Transmittance Update:** Update `log_transmit` based on density and step size.
    *   **Termination:** If the transmittance drops below a threshold, terminate the ray:
        $$ \text{if } \exp(\log\_transmit) < opt.stop\_thresh \text{, break} $$
    *   **Sparsity Regularization (Optional):** If `sparsity_loss > 0.0`, add a regularization term to the density gradient `curr_grad_sigma`. Two variants are mentioned:
        *   Cauchy version (from SNeRG):
            $$ curr\_grad\_sigma += sparsity\_loss \times \frac{4 \times \sigma}{1 + 2 \times (\sigma \times \sigma)} $$
        *   Alphs version (from PlenOctrees):
            $$ curr\_grad\_sigma += sparsity\_loss \times \exp(-pcnt) \times ray.world\_step $$
        [S1, unofficial]

4.  **Gradient Computation (Backward Pass):**
    *   **Sigma Gradient:** Compute the gradient for density (`curr_grad_sigma`) using the chain rule involving accumulated color, transmittance, and step size:
        $$ curr\_grad\_sigma = ray.world\_step \times (invr\_last - invr\_mid) \times (total\_color \times \exp(\log\_transmit) - accum) $$
        Where `invr_last` and `invr_mid` are inverse ray positions, and `accum` is the accumulated transmittance/color term [S1, unofficial].
    *   **Gradient Normalization (if `grad_out_is_rgb`):** If the gradient output is RGB, normalize the residual:
        $$ norm\_factor = \frac{2.0}{3 \times N_{rays}} $$
        $$ grad\_out[i] = resid \times norm\_factor $$
        Otherwise, copy `grad_output` directly [S1, unofficial].
    *   **Propagate Gradients:** Update `PackedGridOutputGrads` using `trilerp_backward_bg_one`. This function updates `grad_background_out` and optionally `mask_background_out` based on `curr_grad_sigma` (or `curr_grad_color` for color gradients) [S1, unofficial].

**Parameters:**
*   `sparsity_loss`: Scalar weight for sparsity regularization. Typical values are $> 0.0$ to encourage transparency in background regions [S1, unofficial].
*   `stop_thresh`: Transmittance threshold for early ray termination. Expressed as a value for `log_transmit` in the check $\exp(\log\_transmit) < opt.stop\_thresh$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_background_forward(ray, grid, opt):
    """
    Forward pass for background volume rendering.
    ray: object with .origin, .direction, .world_step, .find_bounds()
    grid: object with .background_links, .background_data
    opt: object with .stop_thresh, .sparsity_loss
    """
    # 1. Find bounds
    bounds = ray.find_bounds_bg(grid)
    if bounds is None:
        return 0.0, 0.0  # No intersection

    # 2. Initialize accumulation
    total_color = 0.0
    log_transmit = 0.0  # log(1.0) = 0
    accum = 0.0
    curr_grad_sigma = 0.0

    # 3. Ray march
    # Note: Simplified marching loop; actual implementation depends on grid traversal
    for sample in ray.march(bounds):
        # Sample density and color from background grid
        sigma, color = grid.sample_background(sample.position)
        
        # Clamp color
        color = max(color, 0.0)
        
        # Compute weight (transmission factor)
        # weight = exp(-sigma * step) * current_transmittance
        # In log space: log_weight = -sigma * step + log_transmit
        step = ray.world_step
        log_weight = -sigma * step + log_transmit
        weight = math.exp(log_weight)
        
        # Accumulate color
        total_color += color * weight
        
        # Update transmittance
        log_transmit = log_weight
        
        # Termination check
        if math.exp(log_transmit) < opt.stop_thresh:
            break
            
        # Sparsity regularization (if needed, usually for backward pass)
        if opt.sparsity_loss > 0.0:
            # Cauchy version
            curr_grad_sigma += opt.sparsity_loss * (4.0 * sigma / (1.0 + 2.0 * (sigma * sigma)))
            # Or Alphs version:
            # curr_grad_sigma += opt.sparsity_loss * math.exp(-pcnt) * step

    return total_color, math.exp(log_transmit)

def render_background_backward(ray, grid, grad_output, opt, grad_out_is_rgb=False):
    """
    Backward pass for background volume rendering.
    """
    # 1. Skip rays with very low transmittance
    # Assume log_transmit is computed or passed in; if not, compute forward first
    # For efficiency, this check is often done in the kernel launch or early in forward
    if ray.log_transmit < -25.0:
        return PackedGridOutputGrads()

    # 2. Normalize gradient if RGB
    if grad_out_is_rgb:
        N_rays = len(ray.origins)
        norm_factor = 2.0 / (3.0 * N_rays)
        grad_output = grad_output * norm_factor

    # 3. Compute gradients
    # This requires the forward pass values (total_color, log_transmit, accum, etc.)
    # Assuming these are available from the forward pass or recomputed
    # curr_grad_sigma formula from source:
    # curr_grad_sigma = ray.world_step * (invr_last - invr_mid) * (total_color * exp(log_transmit) - accum)
    
    # Note: invr_last and invr_mid are inverse ray positions at the last and mid steps
    # This is a simplified representation; actual implementation needs precise ray state tracking
    
    # 4. Propagate gradients to grid
    grads = PackedGridOutputGrads()
    trilerp_backward_bg_one(grid, ray, curr_grad_sigma, grad_output, grads)
    
    return grads

class PackedGridOutputGrads:
    def __init__(self):
        self.grad_background_out = None
        self.mask_background_out = None
```

**Edge Cases and Pitfalls:**
*   **Low Transmittance Rays:** Rays with `log_transmit < -25.0` are skipped. This assumes negligible contribution but may miss subtle background details in very transparent regions [S1, unofficial].
*   **Variable Name Reuse:** The source snippet uses `grad_output` in the forward accumulation formula, which is likely a misnomer for weight/transmission. Implementers should use standard volume rendering accumulation `weight * color` [S1, unofficial].
*   **Sparsity Regularization:** The choice between Cauchy and Alphs versions depends on the specific background representation (SNeRG vs. PlenOctrees) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of ray samples within the background grid bounds. Early termination based on `stop_thresh` reduces average case complexity. Skipping rays with `log_transmit < -25.0` further optimizes performance [S1, unofficial].
*   **Memory Complexity:** Proportional to the size of the background sparse grid (`grid.background_data`, `grid.background_links`). Separate from foreground grid memory [S1, unofficial].
*   **Strengths:** Efficient rendering of unbounded backgrounds; separate resolution/sampling for background vs. foreground; early termination and ray skipping optimize performance [S1, unofficial].
*   **Limitations:** Requires separate grid management; sparsity regularization parameters need tuning; low transmittance ray skipping may lose detail [S1, unofficial].
*   **When to Use:** When rendering scenes with significant distant or unbounded background elements that benefit from sparse representation [S1, unofficial].
*   **When Not to Use:** For tightly bounded scenes where background is negligible or fully contained in the foreground grid [S1, unofficial].

## Background
This method is part of the JNeRF project, an unofficial implementation of NeRF using the Jittor framework [S1, unofficial]. It extends standard volume rendering by introducing a separate sparse grid for background elements, allowing for more flexible and efficient handling of large-scale scenes [S1, unofficial].

## Sources
[S1] Jittor/JNeRF, https://github.com/Jittor/JNeRF, fetched 2026-10-27 (unofficial)
