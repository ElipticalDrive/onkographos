# Soft Shadows via Area Lights
Also called: Area Light Shadow Sampling, Area Light Shadows, Partial Occlusion, Shadow Occlusion
## Overview
This method computes soft shadows by modeling light sources with spatial extent, allowing points in space to be partially occluded rather than fully lit or fully shadowed [S2, unofficial]. Unlike point source approximations that cast sharp shadows, this approach generates smooth penumbra transitions by sampling the light source area [S1, unofficial]. It differs from its sibling [Soft Shadows via Ray Tracing](5136-soft-shadows-via-ray-tracing.md) by focusing on the area sampling mechanism itself, and from [Point Light Shadow Shader](13086-point-light-shadow-shader.md) by avoiding post-process Gaussian blurring in favor of direct area integration [S1, unofficial].
## How it works
**Inputs:**
*   Volume density field $V(\mathbf{x})$ or distance field $D(\mathbf{x})$.
*   Light source geometry (area definition).
*   Evaluation point $\mathbf{p}$.

**Outputs:**
*   Shadow factor $S \in [0, 1]$ representing the fraction of the light source visible from $\mathbf{p}$.

**Data Structures:**
*   Light source samples: A set of $N$ points $\{\mathbf{l}_i\}_{i=1}^N$ distributed over the light area.
*   Ray step size $\Delta s$ (inherited from parent [Shadow Ray Casting](2796-shadow-ray-casting.md)).

**Steps:**
1.  **Sample Light Area:** Generate $N$ sample points $\mathbf{l}_i$ uniformly or according to a distribution over the light source's geometric area.
2.  **Initialize Accumulator:** Set $visible\_count = 0$.
3.  **Iterate Samples:** For each light sample $\mathbf{l}_i$:
    a. Construct a shadow ray from $\mathbf{p}$ to $\mathbf{l}_i$.
    b. Perform visibility test along this ray using the parent method's transmittance calculation [G].
    c. If the ray reaches $\mathbf{l}_i$ without significant occlusion (transmittance $T \approx 1$), increment $visible\_count$.
4.  **Compute Shadow Factor:** Calculate the ratio of visible samples:
    $$ S = \frac{visible\_count}{N} $$
5.  **Apply Shadowing:** Modulate the direct illumination contribution by $S$.

**Parameters:**
*   $N$: Number of shadow rays. Typical values range from 4 to 64 for real-time, higher for offline [S1, unofficial].

**Edge Cases:**
*   **Small $N$:** Results in noisy, aliased shadows.
*   **Large $N$:** Computationally expensive, diminishing returns [S1, unofficial].
*   **Degenerate Light:** If the light area is zero, it behaves as a point light (hard shadow).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_area_light_shadow(p, light_samples, volume_sampler):
    """
    Computes soft shadow factor for point p given samples from an area light.
    
    Args:
        p: 3D vector, evaluation point.
        light_samples: List of 3D vectors, points on the light source.
        volume_sampler: Function that returns transmittance along a ray from p to target.
        
    Returns:
        float: Shadow factor between 0.0 (fully shadowed) and 1.0 (fully lit).
    """
    visible_count = 0
    n_samples = len(light_samples)
    
    for l in light_samples:
        # Use parent method's visibility check
        # Assuming volume_sampler returns transmittance T from p to l
        # T = 1.0 means no occlusion, T = 0.0 means fully occluded
        transmittance = volume_sampler.sample_transmittance(p, l)
        
        # Threshold for "visible" to handle partial occlusion along the ray
        # If transmittance is above a threshold, count as visible
        if transmittance > 0.5:
            visible_count += 1
            
    return visible_count / n_samples
```
## Subtopics

### [Soft Shadows via Ray Tracing](5136-soft-shadows-via-ray-tracing.md)
Generates coherent packets of secondary shadow rays from primary hit points toward an area light. It differs by managing ray coherence and interactive performance-quality trade-offs via ray count control.

### [Point Light Shadow Shader](13086-point-light-shadow-shader.md)
Applies shadow rays to a point light with plane intersection refinement and Gaussian filtering. It differs by modeling a single point source and using post-process blur rather than direct area sampling.

## Operating characteristics
**Time Complexity:** $O(N \cdot M)$, where $N$ is the number of light samples and $M$ is the number of steps per shadow ray [S1, unofficial].
**Memory Complexity:** $O(1)$ auxiliary memory beyond the light samples [G].
**Performance:** Rendering performance decreases exponentially with the number of rays if not optimized [S1, unofficial].
**Strengths:** Produces natural-looking soft shadows with penumbra; works with dynamic geometry in raymarching contexts [S2, unofficial].
**Limitations:** High computational cost for high-quality results; noisy with low sample counts [S1, unofficial].
**When to use:** When realistic soft shadows are required and computational budget allows for multiple shadow rays per pixel.
**When not to use:** When hard shadows are acceptable or performance is critical with limited ray budget.
## Background
The concept of modeling light sources with spatial extent to achieve soft shadows originated in computer graphics as a departure from point source approximations [S2, unofficial]. In raster engines, this was traditionally achieved via distance-based low-pass filtering on shadow maps [S2, unofficial]. In raymarching-based renderers, it is possible to use distance fields or volume density to approximate soft shadows directly by sampling the light area [S2, unofficial]. The method was adapted for volume graphics by shifting the problem to shooting multiple rays per intersection point to achieve realistic images with minimal ray counts [S1, unofficial].
## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
