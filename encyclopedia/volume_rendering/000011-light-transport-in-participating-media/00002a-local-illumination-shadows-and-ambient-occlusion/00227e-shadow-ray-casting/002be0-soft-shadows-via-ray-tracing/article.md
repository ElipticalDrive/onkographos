# Soft Shadows via Ray Tracing
Also called: Packet-based Soft Shadow Ray Tracing

## Overview
This method computes soft shadows in participating media by generating coherent packets of secondary rays from primary ray hit points toward samples on an area light source [S1]. It distinguishes itself from hard shadow techniques by sampling a spatial distribution of directions to approximate the penumbra, leveraging the coherence of these secondary rays to maintain interactive performance [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Primary ray hit point $\mathbf{p}$, primary ray direction $\boldsymbol{\omega}$, area light source geometry, medium properties ($\sigma_t$).
*   **Output:** Transmittance value $T$ representing the fraction of light reaching the point from the area light.

**Data Structures**
*   **Shadow Ray Packet:** A collection of $N$ shadow rays sharing a common origin $\mathbf{p}$. Each ray $i$ has direction $\boldsymbol{\omega}_i$ pointing toward a sample $\mathbf{l}_i$ on the area light source.
*   **Light Samples:** A set of points $\{\mathbf{l}_1, \dots, \mathbf{l}_N\}$ distributed over the area light source surface.

**Steps**
1.  **Generate Light Samples:** Select $N$ points $\mathbf{l}_i$ on the area light source. The number $N$ and the spatial extent of the light source are user-controlled parameters [S1].
2.  **Construct Shadow Ray Packet:** For each light sample $\mathbf{l}_i$, compute the direction $\boldsymbol{\omega}_i = \text{normalize}(\mathbf{l}_i - \mathbf{p})$. Form a packet of $N$ rays originating at $\mathbf{p}$ with directions $\{\boldsymbol{\omega}_1, \dots, \boldsymbol{\omega}_N\}$ [S1].
3.  **Traverse Medium:** Traverse the medium along each ray in the packet. Since all rays share the origin $\mathbf{p}$, they can be traversed using the same coherent grid traversal logic as the primary rays [S1].
4.  **Compute Transmittance:** For each ray $i$, calculate the transmittance $T_i$ through the medium using the Beer-Lambert law:
    $$ T_i = e^{-\int_{0}^{d_i} \sigma_t(\mathbf{p} + t\boldsymbol{\omega}_i) dt} $$
    where $d_i$ is the distance to the light sample or the first occluder [S3, unofficial].
5.  **Aggregate:** The final shadow factor is the average of the transmittances:
    $$ T_{\text{shadow}} = \frac{1}{N} \sum_{i=1}^{N} T_i $$

**Parameters**
*   $N$: Number of shadow rays per packet. Controlled interactively by the user [S1].
*   Light Source Size: The spatial extent of the area light. Controlled interactively by the user [S1].

**Edge Cases**
*   **Degenerate Light:** If the light source has zero area, $N$ should be 1, reducing to hard shadow casting.
*   **Precision:** Small distances between $\mathbf{p}$ and $\mathbf{l}_i$ may cause numerical instability in direction normalization. [G] Add a small epsilon to the denominator.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_soft_shadow(p, light_samples, medium_traverser):
    """
    Compute soft shadow transmittance for a point p using a packet of shadow rays.
    
    Args:
        p: Hit point position (vec3).
        light_samples: List of N light source positions (list of vec3).
        medium_traverser: Object capable of traversing the medium and returning transmittance.
        
    Returns:
        float: Average transmittance (shadow factor).
    """
    n = len(light_samples)
    if n == 0:
        return 1.0  # No light source, fully lit (or undefined, depending on context)
    
    total_transmittance = 0.0
    
    for light_pos in light_samples:
        # Compute direction from hit point to light sample
        direction = normalize(light_pos - p)
        
        # Traverse the medium along this direction
        # medium_traverser handles the coherent traversal logic
        # It returns the transmittance T_i for this specific ray
        t_i = medium_traverser.traverse(p, direction, light_pos)
        
        total_transmittance += t_i
        
    return total_transmittance / n
```

## Subtopics

### [Soft Shadows via Ray Packets](5179-soft-shadows-via-ray-packets.md)
This method extends ray casting in participating media to support non-local illumination effects, specifically soft shadows, by tracing secondary ray packets from primary ray origins toward area light sources. It distinguishes itself from hard shadow techniques by sampling a spatial distribution of directions to approximate the penumbra, leveraging the coherence of these secondary rays to maintain interactive performance.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of shadow rays per packet and $K$ is the average number of grid cells traversed per ray. Since rays are coherent, traversal overhead is minimized compared to independent rays.
*   **Memory Complexity:** $O(N)$ for storing the packet directions and intermediate traversal states.
*   **Performance:** Interactive performance is achievable because the number of shadow rays $N$ and the light source size can be controlled by the user [S1]. Higher $N$ improves shadow quality (softer penumbra) but increases computational cost linearly.
*   **Strengths:** Naturally handles soft shadows in participating media; coherent traversal reduces overhead; user-controllable quality/performance trade-off.
*   **Limitations:** Requires area light sampling; performance degrades with very large light sources or high $N$; not suitable for dynamic scenes with rapidly changing light positions without re-traversal.

## Background
The method was introduced in the context of coherent grid traversal for particle visualization, where shadows and global effects are easily integrated into ray tracing approaches [S1]. It leverages the coherence of shadow rays sharing a common origin to optimize traversal, similar to primary ray packets.

## Sources
[S1] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
