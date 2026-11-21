# Partial Segment Handling

Also called: Reduced Segment Length Opacity Adjustment, Segmental Replacement, Segmental Replacer Node

## Overview
Partial Segment Handling addresses the "jerky edges" or stair-stepping artifacts that occur at volume boundaries or cutting surfaces when rays are sampled at fixed intervals, leaving an incomplete segment at the end of the ray march [S1, unofficial]. It solves this by introducing a smaller final segment to fill the remaining distance to the boundary and adjusting its opacity to preserve the correct accumulated attenuation [S1, unofficial]. A variant, Segmental Replacement, uses surface models to partially replace volume models to exhibit internal information while avoiding the rendering of the entire volume, facilitating real-time simulation [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Original segment length $d_1$ (typically the fixed step size $\Delta t$) [S1, unofficial].
*   Original segment opacity $\alpha_1$ (derived from the transfer function and extinction coefficient for length $d_1$) [S1, unofficial].
*   Remaining distance to the volume boundary or cutting surface $d_2$ (where $0 < d_2 < d_1$) [S1, unofficial].
*   Optional surface model data for the replacement region [S2, unofficial].

**Outputs:**
*   Adjusted opacity $\alpha_2$ for the partial segment [S1, unofficial].
*   Rendered pixel color using either the adjusted volume segment or the surface counterpart [S2, unofficial].

### Data Structures
No specific complex data structures are required beyond the standard ray marching state [G]. The method operates on scalar values associated with the current ray step [G].
*   **Segment State:** Contains current position $t$, accumulated color $\mathbf{C}_{acc}$, and accumulated opacity $\alpha_{acc}$ [G].
*   **Boundary Check:** A boolean or distance value indicating if the next full step $t + d_1$ exceeds the volume exit $t_{far}$ or a cutting plane [G].

### Steps
1.  **Ray Marching:** Perform standard front-to-back compositing for full segments of length $d_1$ as defined in [Direct volume ray casting](19-direct-volume-ray-casting.md) [G].
2.  **Boundary Detection:** At each step, check if the next sample position $t_{next} = t_{current} + d_1$ exceeds the volume exit parameter $t_{far}$ or intersects a cutting plane [G].
3.  **Partial Segment Calculation:** If a boundary is crossed:
    *   Calculate the remaining distance $d_2 = t_{far} - t_{current}$ (or distance to cutting plane) [S1, unofficial].
    *   Retrieve the volumetric data value at $t_{current}$ [G].
    *   Determine the original opacity $\alpha_1$ that would have been applied if the segment length were $d_1$ [S1, unofficial].
    *   Calculate the adjusted opacity $\alpha_2$ for the shorter length $d_2$ using the formula below [S1, unofficial].
    *   Composite the partial segment using $\alpha_2$ and the color $\mathbf{c}$ derived from the data at $t_{current}$ [G].
    *   Terminate the ray march [G].
4.  **Segmental Replacement (Variant):** If using the Segmental Replacer Node strategy:
    *   Clip the volume model along vertical and horizontal planes to generate quadruple sub-parts of the original model [S2, unofficial].
    *   Identify regions where the volume model is replaced by a surface counterpart [S2, unofficial].
    *   Render the surface counterpart for the darker areas to exhibit internal information while avoiding the cost of rendering the entire volume model [S2, unofficial].

### Formulas
The core adjustment for reduced segment length is [S1, unofficial]:

$$
\alpha_2 = 1 - (1 - \alpha_1)^{\frac{d_2}{d_1}}
$$

Where:
*   $\alpha_1$ is the opacity of the original full segment of length $d_1$ [S1, unofficial].
*   $d_1$ is the original segment length (step size) [S1, unofficial].
*   $d_2$ is the new, reduced segment length (remaining distance to boundary) [S1, unofficial].
*   $\alpha_2$ is the adjusted opacity for the partial segment [S1, unofficial].

This formula ensures that the transmission $(1 - \alpha_2)$ equals the transmission of the original segment raised to the power of the length ratio, preserving the Beer-Lambert law's exponential attenuation property for the partial distance [S1, unofficial].

### Parameters
*   **Segment Length ($d_1$):** Typically the fixed step size $\Delta t$ used in the ray march. Adaptive strategies may vary $d_1$ based on contribution [S1, unofficial].
*   **Transfer Function:** Maps scalar values to extinction coefficient $\sigma$ and color $\mathbf{c}$ [G].

### Edge Cases and Pitfalls
*   **Zero Remaining Distance:** If $d_2 \approx 0$, $\alpha_2$ should be 0 to avoid division by zero or numerical instability in the exponent [G].
*   **Precision:** Ensure floating-point precision is sufficient for the exponentiation, especially when $d_2 \ll d_1$ [G].
*   **Adaptive Segments:** Smaller segments are used when the contribution to the ray color is large, and longer segments when the contribution is low, to optimize for speed while yielding few artifacts [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def partial_segment_handling(ray_origin, ray_direction, t_near, t_far, step_size, transfer_function, volume_grid, grid_spacing, grid_origin):
    """
    Renders a ray with partial segment handling at boundaries.
    Implements standard front-to-back compositing [G] with opacity adjustment [S1, unofficial].
    """
    # Standard trilinear interpolation helper [G]
    def trilinear_interpolate(pos):
        gx = (pos[0] - grid_origin[0]) / grid_spacing[0]
        gy = (pos[1] - grid_origin[1]) / grid_spacing[1]
        gz = (pos[2] - grid_origin[2]) / grid_spacing[2]
        gx = max(0, min(gx, volume_grid.shape[0] - 1))
        gy = max(0, min(gy, volume_grid.shape[1] - 1))
        gz = max(0, min(gz, volume_grid.shape[2] - 1))
        # Bilinear/trilinear interpolation logic omitted for brevity, returns scalar value [G]
        return volume_grid[int(gx), int(gy), int(gz)]

    t = t_near
    acc_color = (0.0, 0.0, 0.0)
    acc_opacity = 0.0

    while t < t_far:
        t_next = t + step_size
        d1 = step_size
        is_partial = t_next > t_far
        d2 = t_far - t if is_partial else d1

        t_sample = t + d2 / 2.0
        val = trilinear_interpolate(ray_origin + t_sample * ray_direction)
        color, sigma = transfer_function(val)

        # Compute original opacity for full step [S1, unofficial]
        alpha1 = 1.0 - math.exp(-sigma * d1)
        # Compute adjusted opacity for partial step using source formula [S1, unofficial]
        alpha2 = 1.0 - math.pow((1.0 - alpha1), d2 / d1)
        alpha = alpha2 if is_partial else alpha1

        # Front-to-back compositing [G]
        weight = (1.0 - acc_opacity) * alpha
        acc_color = (
            acc_color[0] + weight * color[0],
            acc_color[1] + weight * color[1],
            acc_color[2] + weight * color[2]
        )
        acc_opacity = acc_opacity + weight

        t = t_next

    return acc_color, acc_opacity
```

## Subtopics

### Segmental Replacement
A strategy where a surface model partially replaces a volume model to exhibit internal information while avoiding the rendering of the entire volume model. It is chosen when real-time simulation requires avoiding the cost of full volume rendering, and the trade-off is acceptable [S2, unofficial].

### Adaptive Segment Length
An optimization where smaller segments are used when the contribution to the ray color is large, and longer segments when the contribution is low. This reduces artifacts while optimizing for speed [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples. The partial segment handling adds a constant-time check and calculation at the end of the ray march [G].
*   **Memory Complexity:** $O(1)$ additional memory. No new data structures are required [G].
*   **Performance:** The method is computationally inexpensive, adding minimal overhead to the ray march. It is suitable for real-time applications [S1, unofficial].
*   **Strengths:** Eliminates jerky edges at boundaries without requiring global resampling or increased sampling density [S1, unofficial].
*   **Limitations:** Only addresses artifacts at the end of the ray march [S1, unofficial].

## Background
Partial Segment Handling was identified as a solution to common artifacts in volume rendering, specifically jerky edges caused by incomplete segments at volume boundaries [S1, unofficial]. The technique is based on the Beer-Lambert law's exponential attenuation properties [S1, unofficial]. The concept of using surface models to partially replace volume models (Segmental Replacement) was explored in hybrid rendering frameworks for real-time manipulation of volume and surface models [S2, unofficial].

## Sources
[S1] Common Artifacts in Volume Rendering, https://doi.org/10.48550/arxiv.2109.13704, fetched 2026-10-26 (unofficial)
[S2] A Hybrid Rendering Framework for the Real-time Manipulation of Volume and Surface Models, https://core.ac.uk/download/55863.pdf, fetched 2026-11-03 (unofficial)
