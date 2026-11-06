# Soft Shadows via Area Lights

Also called: Area Light Shadow Sampling, Area Light Shadows, Partial Occlusion, Shadow Occlusion

## Overview
This method models shadows cast by light sources with spatial extent, allowing points in space to be partially occluded, unlike point sources which cast sharp shadows [S2, unofficial]. It differs from its sibling [Direct Illumination with Shadow Rays](2011-direct-illumination-with-shadow-rays.md) by replacing the single boolean visibility test with multiple samples across the light's area to compute natural soft shadows with penumbra [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
*   Volume density field $\sigma_t(\mathbf{x})$.
*   Light source geometry defining an area $A$ and position $\mathbf{p}_L$.
*   Evaluation point $\mathbf{p}$.
*   Number of samples $N$.

**Outputs:**
*   Soft shadow factor $S \in [0, 1]$, representing the fraction of the light source visible from $\mathbf{p}$.

**Data Structures:**
*   Sample points $\mathbf{l}_i$ distributed over the light area $A$.

**Steps:**
1.  Sample $N$ points $\mathbf{l}_i$ uniformly (or via a pattern) across the light source area $A$.
2.  For each sample $i$, compute the direction vector $\mathbf{d}_i = \mathbf{l}_i - \mathbf{p}$.
3.  Perform a shadow ray cast from $\mathbf{p}$ along $\mathbf{d}_i$ to determine visibility $v_i$.
    *   If the ray hits an occluder (density threshold exceeded) before reaching $\mathbf{l}_i$, $v_i = 0$.
    *   Otherwise, $v_i = 1$.
4.  Compute the soft shadow factor as the average visibility:
    $$ S = \frac{1}{N} \sum_{i=1}^{N} v_i $$

**Parameters:**
*   $N$: Number of samples. Higher values reduce noise but increase cost. Not given in the sources.

**Edge Cases:**
*   If $N=1$, the method degenerates to hard shadow casting [S2, unofficial].
*   Precision issues may arise if $\mathbf{p}$ is extremely close to the light source or occluder.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_soft_shadow(p, light_area, sigma_t, N):
    """
    Compute soft shadow factor for point p given light_area and volume sigma_t.
    light_area: object with method sample(N) returning N points on the light surface.
    sigma_t: function returning density at a point.
    """
    samples = light_area.sample(N)
    visible_count = 0
    
    for l_point in samples:
        direction = l_point - p
        # Simplified shadow ray cast: check if any point along the ray is occluded
        # In a full implementation, this would involve raymarching
        is_visible = True
        t = 0.0
        step_size = 0.01 # Not given in the sources.
        max_dist = length(direction)
        
        while t < max_dist:
            current_pos = p + direction * (t / max_dist)
            if sigma_t(current_pos) > 0.1: # Threshold not given in the sources.
                is_visible = False
                break
            t += step_size
            
        if is_visible:
            visible_count += 1
            
    return visible_count / N
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of light samples and $M$ is the number of steps per shadow ray.
*   **Memory Complexity:** $O(1)$ additional memory beyond the volume data.
*   **Strengths:** Produces physically plausible soft shadows with penumbra.
*   **Limitations:** Computationally expensive due to multiple ray casts per shading point. Performance degrades exponentially with increasing $N$ [S1, unofficial].
*   **When to use:** When realistic soft shadows are required and performance allows for multiple ray samples.
*   **When not to use:** For real-time applications with strict performance constraints or when hard shadows are sufficient.

## Background
The concept of modeling light sources with spatial extent to achieve soft shadows was discussed in the context of raymarching distance fields [S2, unofficial]. The application of area light sampling to volume graphics for more natural appearing soft shadows was described in the context of interactive volume ray tracing [S1, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
