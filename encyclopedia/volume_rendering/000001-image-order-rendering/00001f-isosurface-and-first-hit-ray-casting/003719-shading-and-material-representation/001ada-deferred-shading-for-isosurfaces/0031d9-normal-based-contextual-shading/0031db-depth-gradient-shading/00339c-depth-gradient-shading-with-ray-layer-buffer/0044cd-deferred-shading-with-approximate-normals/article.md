# Deferred Shading with Approximate Normals

Also called: Accumulated Gradient Shading, Deferred Shading with Layer Separation Heuristic

## Overview
This method calculates view-dependent lighting in layered volume rendering by storing a single representative normal vector per ray segment and performing shading in a deferred step before compositing, rather than multiplying shading coefficients with fragment color before warping [S1]. It solves the problem of noisy shading in regions with low volume gradient magnitudes by using a weighted accumulation strategy that prioritizes samples with high visual impact [S1]. It differs from its parent, [Depth-Gradient Shading with Ray-Layer Buffer](7010-depth-gradient-shading-with-ray-layer-buffer.md), by explicitly storing and shading with normal vectors derived from volume gradients rather than depth gradients, allowing for more accurate specular highlights and surface perception [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset $V$, transfer function $T$, camera parameters, ray-layer buffer containing depth and opacity information.
*   **Output:** Final composited image with view-dependent lighting.

**Data Structures:**
*   **Ray Segment:** A contiguous range of samples along a ray between two layer boundaries.
*   **Accumulated Normal:** A 3D vector $\mathbf{N}_{acc}$ initialized to $(0,0,0)$.
*   **Accumulated Alpha:** A scalar $\alpha_{acc}$ initialized to $0$.

**Numbered Steps:**
1.  **Ray Traversal:** For each pixel, cast a ray through the volume. Identify the ray segments defined by the layer boundaries.
2.  **Gradient Computation:** At each sample point $i$ within a segment, compute the volume gradient $\nabla V_i$ [S1].
3.  **Normalization:** Normalize the gradient to obtain the surface normal direction: $\mathbf{n}_i = \frac{\nabla V_i}{\|\nabla V_i\|}$ [S1]. If $\|\nabla V_i\| \approx 0$, skip or handle as noise [G].
4.  **Weighted Accumulation:** Accumulate the normals weighted by visual impact. The weight $w_i$ for sample $i$ is calculated as:
    $$ w_i = \alpha_i \cdot (1 - \alpha_{prev\_i})^3 $$
    where $\alpha_i$ is the opacity contribution of sample $i$, and $(1 - \alpha_{prev\_i})$ is the remaining opacity (transmittance) from the camera to sample $i$ [S1]. The remaining opacity is raised to the third power to weight closer samples more strongly [S1].
5.  **Final Normal Selection:** Compute the accumulated normal:
    $$ \mathbf{N}_{acc} = \sum_{i \in \text{segment}} w_i \cdot \mathbf{n}_i $$
    Normalize the result: $\mathbf{N}_{final} = \frac{\mathbf{N}_{acc}}{\|\mathbf{N}_{acc}\|}$ [S1]. If $\|\mathbf{N}_{acc}\| \approx 0$, use a default normal (e.g., view direction or zero) [G].
6.  **Deferred Shading:** After all rays are processed and layers are composited, use $\mathbf{N}_{final}$ for each fragment to calculate lighting (diffuse/specular) in a post-processing pass [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_accumulated_normal(samples, remaining_opacity):
    """
    Computes the accumulated normal for a ray segment.
    samples: list of dicts with keys 'gradient' (vec3), 'alpha' (float)
    remaining_opacity: float, transmittance from camera to start of segment
    """
    acc_normal = vec3(0, 0, 0)
    acc_alpha = 0.0
    
    for sample in samples:
        grad = sample['gradient']
        alpha = sample['alpha']
        
        # Normalize gradient to get normal direction
        grad_norm = length(grad)
        if grad_norm < 1e-6:
            continue # Skip zero gradients
        
        normal = grad / grad_norm
        
        # Calculate weight: alpha * (remaining_opacity)^3
        # remaining_opacity is updated per sample in traversal
        weight = alpha * (remaining_opacity ** 3)
        
        acc_normal += weight * normal
        acc_alpha += weight
        
        # Update remaining opacity for next sample
        remaining_opacity *= (1.0 - alpha)
        
    # Normalize the accumulated normal
    acc_norm_len = length(acc_normal)
    if acc_norm_len < 1e-6:
        return vec3(0, 0, 0) # Fallback for degenerate case
        
    return acc_normal / acc_norm_len
```

**Parameters:**
*   **Opacity Power:** The exponent applied to remaining opacity. The source specifies a value of **3** [S1].
*   **Gradient Smoothing:** Not specified in sources. Noise in CT/MRT data may require pre-smoothing [S1].

**Edge Cases and Pitfalls:**
*   **Noisy Data:** CT or MRT datasets often have noisy measured data, leading to noisy gradients [S1]. Choosing the wrong gradient causes strong visual differences compared to per-sample shaded raycasting [S1].
*   **Zero Gradient:** If the volume gradient is zero, the normal is undefined. A fallback is required [G].
*   **Strategy Selection:** Simple strategies like selecting the longest gradient or the gradient at opacity maximum perform worse than the accumulated method [S1].

## Subtopics

### Longest Gradient Selection
Selects the sample with the maximum gradient magnitude as the representative normal. Simpler but less accurate [S1].

### Gradient at Opacity Maximum
Selects the gradient at the sample with the highest opacity contribution. Intuitive but suboptimal for view-dependent lighting [S1].

### Standard Accumulated Gradient
Accumulates normals weighted by $\alpha \cdot (1-\alpha_{prev})$. Less effective than the cubic opacity weighting for handling noise [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples per segment. The deferred shading step is $O(P)$ where $P$ is the number of pixels.
*   **Memory Complexity:** Requires storing one normal vector per fragment in addition to standard layer buffers.
*   **Performance:** Evaluation on MANIX and VIX datasets showed that the 'accumulated w. opacity^3' method provides the best visual quality (lowest average absolute pixel error) across rotation angles [S1].
*   **Strengths:** Reduces noise in shading compared to per-sample shading; provides view-dependent lighting in a layered rendering context.
*   **Limitations:** Approximate normal; may not perfectly match per-sample shaded raycasting; sensitive to gradient noise in raw data [S1].

## Background
This method was introduced by S. Hauswiesner, D. Kalkofen, and D. Schmalstieg in "Multi-Frame Rate Volume Rendering" (2010) [S1]. It addresses the challenge of performing view-dependent lighting in layered volume rendering, where traditional per-sample shading is not directly applicable due to the layering heuristic [S1].

## Sources
[S1] Multi-Frame Rate Volume Rendering, https://doi.org/10.2312/egpgv.egpgv10.019-026, fetched 2026-10-29
