# Background Rendering Kernel (Forward)
## Overview
A CUDA kernel that renders the background color for rays that have transmitted through the volume [S1, unofficial]. It composites the background environment onto the accumulated volume color for rays that have not been fully occluded.
## How it works
**Inputs and Outputs:**
- `grid`: Background environment data structure.
- `ray_spec`: Ray origins and directions.
- `options`: Rendering configuration parameters.
- `log_transmit`: Per-ray logarithmic transmission values.
- `output`: Buffer for the resulting background color.

**Data Structures:**
- `log_transmit`: Array of floats, indexed by `ray_id`. Stores the natural logarithm of the accumulated transmission along the ray through the volume [S1, unofficial].

**Steps:**
1. **Initialize Ray Specification**: Extract ray origins and directions from the input buffers for the current thread's `ray_id` [S1, unofficial].
2. **Find Background Bounds**: Call `ray_find_bounds_bg` to determine the intersection interval of the ray with the background environment bounds [S1, unofficial].
3. **Check Occlusion**: If `log_transmit[ray_id] < -25.0`, the ray is considered highly occluded and the thread returns early, skipping background rendering [S1, unofficial].
4. **Render Background Forward**: Call `render_background_forward` with the `grid`, `ray_spec`, `options`, `log_transmit`, and `output` buffers [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_background_kernel_forward(grid, ray_origins, ray_directions, options, log_transmit, output):
    ray_id = cuda.grid(1) [G]
    if ray_id >= num_rays:
        return

    origin = ray_origins[ray_id]
    direction = ray_directions[ray_id]
    ray_spec = Ray(origin, direction)

    # Find background bounds: ray_find_bounds_bg is called [S1, unofficial]
    t_min, t_max = ray_find_bounds_bg(ray_spec, grid, options)

    if log_transmit[ray_id] < -25.0:
        return

    # Render background forward: render_background_forward is called [S1, unofficial]
    render_background_forward(grid, ray_spec, options, log_transmit, output, ray_id)
```

**Parameters:**
- `log_transmit` threshold: `-25.0`. Rays with transmission less than $e^{-25}$ are skipped [S1, unofficial].

**Edge Cases and Pitfalls:**
- **High Occlusion**: Rays with very low transmission (log value < -25) are skipped to save computation [S1, unofficial].

## Subtopics

### [Background Rendering via Concentric Spheres](15340-background-rendering-via-concentric-spheres.md)
This method renders the background environment for rays that have transmitted through the volume by sampling a layered spherical map. It differs from standard equirectangular sampling by using concentric spheres to distribute samples, which helps mitigate aliasing and provides a structured traversal of the background layers. It serves as the final compositing stage in the background rendering kernel, accumulating color from the background layers and adding a constant ambient term based on remaining transmission [G].

### Background Rendering Kernel (Backward)
Computes gradients for the forward background rendering pass. It begins by computing `ray_id` and performing an early exit if `log_transmit[ray_id] < -25.0` [S2, unofficial]. It constructs the ray specification and computes background bounds via `ray_find_bounds_bg(ray_spec, grid, opt, ray_id)` [S2, unofficial]. It computes the gradient output: if `grad_out_is_rgb`, `grad_out[i] = (color_cache[ray_id * 3 + i] - grad_output[ray_id * 3 + i]) * (2.0 / (3 * Q))`; otherwise it copies `grad_output` [S2, unofficial]. Finally, it calls the backward background compositing routine `render_background_backward(grid, grad_out, ray_spec, opt, log_transmit[ray_id], accum[ray_id], sparsity_loss, grads)` [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
[S2] OSHMOS/IPRF, https://github.com/OSHMOS/IPRF, fetched 2026-10-27 (unofficial)
