# Monte Carlo Integration for Rendering

Also called: MC Integration, Stochastic Integration

## Overview
Monte Carlo Integration solves the recursive rendering equation integral by converting it into a finite sum of stochastic estimates, allowing for the handling of complex light transport phenomena such as soft shadows, depth-of-field, and self-shadowing [S1, unofficial]. It is distinguished from deterministic volume rendering acceleration methods (e.g., [Early Ray Termination and Culling](19061-early-ray-termination-and-culling.md)) by its reliance on random sampling to approximate integrals rather than spatial culling or structured traversal, making it suitable for general path tracing but computationally intensive due to variance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Image dimensions (width $W$, height $H$), camera parameters (thin lens model), scene geometry/materials, and number of iterations $N$ [S1, unofficial].
*   **Output:** Estimated radiance $C$ for each pixel [S1, unofficial].

**Data Structures:**
*   **Ray Buffer:** A collection of $W \times H$ rays generated per iteration [S1, unofficial].
*   **Accumulator:** Per-pixel sum of estimates [S1, unofficial].

**Algorithm:**
1.  **Initialization:** Initialize pixel accumulators to zero.
2.  **Iteration Loop:** For $i$ from 1 to $N$:
    a.  **Ray Generation:** Generate $W \times H$ camera rays using a thin lens model [S1, unofficial].
    b.  **Light Transport:** Trace rays through the scene, calculating shading and interactions [S1, unofficial].
    c.  **Estimation:** Compute the stochastic estimate $f(X_i)$ for each ray, weighted by the probability density function $p(X_i)$ [S1, unofficial].
    d.  **Accumulation:** Add the weighted estimate to the pixel accumulator.
3.  **Finalization:** Divide the accumulator by $N$ to obtain the final color [S1, unofficial].

**Formula:**
The expected value $C$ is calculated as [S1, unofficial]:
$$ C = \frac{1}{N} \sum_{i=1}^{N} \frac{f(X_i)}{p(X_i)} $$
Where:
*   $N$: Number of iterations (stochastic estimates) per pixel [S1, unofficial].
*   $f(X_i)$: The integrand value (e.g., radiance contribution) for the $i$-th sample [S1, unofficial].
*   $p(X_i)$: The probability density function (PDF) of generating sample $X_i$ [S1, unofficial].
*   $X_i$: The $i$-th stochastic sample (e.g., ray direction, intersection point) [S1, unofficial].

**Parameters:**
*   $N$: Number of iterations. The sources note that increasing $N$ reduces variance and converges to the true value [S1, unofficial]. A specific implementation mentioned 2000 iterations for high-quality results with soft shadows and depth-of-field [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Variance:** Low $N$ results in noisy images. Importance sampling is used to mitigate this [S1, unofficial].
*   **Precision:** Floating-point accumulation errors may occur with very large $N$ or high dynamic range scenes [G].
*   **Degenerate Input:** If $p(X_i) = 0$ for a sample where $f(X_i) \neq 0$, the estimate is undefined [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monte_carlo_render(W, H, N, scene, camera):
    # Initialize accumulators
    color_accumulator = zeros((H, W, 3))
    
    for i in range(N):
        # Generate rays for all pixels
        rays = generate_camera_rays(W, H, camera)
        
        for y in range(H):
            for x in range(W):
                ray = rays[y, x]
                
                # Trace ray and compute integrand value f(X_i)
                # and its probability density p(X_i)
                f_xi, p_xi = trace_ray(ray, scene)
                
                # Avoid division by zero
                if p_xi == 0:
                    contribution = 0.0
                else:
                    contribution = f_xi / p_xi
                
                # Accumulate
                color_accumulator[y, x] += contribution
                
    # Average over N iterations
    final_color = color_accumulator / N
    return final_color
```

## Subtopics

### Reservoir Resampling & Temporal Path Guiding
Reservoir resampling and temporal path guiding are orthogonal variance reduction techniques that exploit temporal, screen space, and scene space correlation [S2]. Both are unbiased, computationally lightweight, and effective at spatio-temporal information reuse, making them suitable for combination with control variates [S2]. They are chosen when spatio-temporal coherence can be leveraged to reduce noise without significant additional computational cost [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot W \cdot H \cdot C)$, where $C$ is the cost per ray (bounces, shading) [S1, unofficial].
*   **Memory Complexity:** $O(W \cdot H)$ for pixel buffers [S1, unofficial].
*   **Performance:** Convergence is slow ($O(1/\sqrt{N})$), requiring many samples (e.g., 2000) for high quality [S1, unofficial].
*   **Strengths:** Handles complex lighting (soft shadows, DOF) naturally; unbiased [S1, unofficial].
*   **Limitations:** High variance; noisy images at low sample counts; computationally expensive [S1, unofficial].
*   **When to Use:** Offline rendering or high-quality stills where noise can be tolerated or denoised [S1, unofficial].
*   **When Not to Use:** Real-time applications requiring low latency without heavy denoising [G].

## Background
Monte Carlo integration for rendering was popularized in the context of global illumination to solve the recursive rendering equation [S1, unofficial]. The approach converts the integral into a finite sum, with each iteration reducing variance [S1, unofficial]. Recent developments include recursive control variates and combinations with temporal path guiding and reservoir resampling for inverse rendering [S2].

## Sources
[S1] A Tile-Based Approach for Photo-Realistic Volume Rendering, http://hdl.handle.net/1794/23836, fetched 2026-10-26 (unofficial)
[S2] Recursive Control Variates for Inverse Rendering, https://doi.org/10.1145/3592139, fetched 2026-10-27
