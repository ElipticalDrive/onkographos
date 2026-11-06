# Soft Shadows via Ray Tracing
Also called: Packet-based Soft Shadow Ray Tracing
## Overview
This method generates soft shadows in participating media by tracing packets of coherent secondary rays from a primary ray's hit point toward an area light source [S1]. It distinguishes itself from hard shadow techniques by sampling a spatial distribution of directions to approximate the penumbra, leveraging the coherence of these secondary rays to maintain interactive performance [S1].
## How it works
**Inputs and Outputs:**
- Input: Primary ray hit point $\mathbf{p}$, area light source geometry, volume density field.
- Output: Shadow factor (transmittance) for the pixel associated with the primary ray.

**Data Structures:**
- **Ray Packet:** A contiguous array of shadow rays sharing a common origin $\mathbf{p}$. Each ray is defined by its direction $\mathbf{d}_i$ and length $L$ (distance to light).
- **Light Samples:** A set of points $\{\mathbf{l}_1, \dots, \mathbf{l}_N\}$ on the area light surface.

**Steps:**
1. **Generate Shadow Ray Directions:** For each sample $\mathbf{l}_i$ on the area light, compute the direction vector $\mathbf{d}_i = \mathbf{l}_i - \mathbf{p}$. Normalize $\mathbf{d}_i$.
2. **Construct Packet:** Create a packet of $N$ shadow rays, all originating at $\mathbf{p}$ with directions $\mathbf{d}_i$ and lengths equal to $\|\mathbf{l}_i - \mathbf{p}\|$.
3. **Traverse Volume:** Traverse the volume data structure (e.g., grid or particle list) using the coherent traversal algorithm for the entire packet [S1]. This exploits spatial coherence between rays in the packet to minimize redundant volume checks.
4. **Evaluate Transmittance:** For each ray $i$ in the packet, compute the optical depth $\tau_i$ by integrating the extinction coefficient $\sigma_t$ along the path from $\mathbf{p}$ to $\mathbf{l}_i$. The transmittance for that sample is $T_i = \exp(-\tau_i)$.
5. **Aggregate Shadow Factor:** The final shadow factor $S$ is the average of the transmittances of all rays in the packet:
   $$ S = \frac{1}{N} \sum_{i=1}^{N} T_i $$
   This average approximates the integral of light over the area light source.

**Parameters:**
- $N$: Number of shadow rays (light samples). Controlled interactively by the user for performance-quality trade-offs [S1].
- Light Source Size: Implicitly controlled by the spread of samples $\mathbf{l}_i$. Larger sources require more samples for smooth penumbra.

**Edge Cases and Pitfalls:**
- **Precision:** If $\mathbf{p}$ is very close to the light source, numerical instability may occur in direction normalization. [G] Clamp minimum distance or use normalized vectors carefully.
- **Degenerate Input:** If the volume is empty, transmittance is 1.0 for all rays.
- **Boundary:** Ensure ray lengths do not exceed the light source distance to avoid sampling outside the light.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_soft_shadow(p, light_samples, volume_traverser):
    """
    Compute soft shadow factor for point p using packet-based ray tracing.
    
    Args:
        p: Hit point (3D vector)
        light_samples: List of N 3D vectors representing points on the area light
        volume_traverser: Object capable of traversing volume and returning transmittance for a ray
        
    Returns:
        shadow_factor: Float between 0 and 1
    """
    n_samples = len(light_samples)
    if n_samples == 0:
        return 1.0
    
    total_transmittance = 0.0
    
    # Step 1 & 2: Generate directions and construct packet implicitly
    # We iterate through samples, computing direction and length for each ray
    for i in range(n_samples):
        l_i = light_samples[i]
        direction = l_i - p
        length = magnitude(direction)
        
        # Avoid division by zero if p is on the light
        if length < 1e-6:
            transmittance = 1.0
        else:
            direction = normalize(direction)
            
            # Step 3 & 4: Traverse volume and evaluate transmittance
            # volume_traverser handles the coherent traversal logic internally
            # It returns the transmittance T_i for the ray from p to l_i
            transmittance = volume_traverser.get_transmittance(p, direction, length)
        
        total_transmittance += transmittance
    
    # Step 5: Aggregate
    shadow_factor = total_transmittance / n_samples
    return shadow_factor
```
## Subtopics

### [Soft Shadows via Ray Packets](5179-soft-shadows-via-ray-packets.md)
This method extends ray casting in participating media to support non-local illumination effects, specifically soft shadows, by tracing secondary ray packets from primary ray origins toward area light sources. It distinguishes itself from hard shadow techniques by sampling a spatial distribution of directions to approximate the penumbra, leveraging the coherence of these secondary rays to maintain interactive performance.

## Operating characteristics
- **Time Complexity:** $O(N \cdot T)$, where $N$ is the number of shadow rays (light samples) and $T$ is the cost of traversing the volume for one ray. Coherent traversal reduces $T$ compared to independent rays.
- **Memory Complexity:** $O(N)$ for storing the packet of rays.
- **Performance:** Interactive performance is achievable due to coherence exploitation [S1]. Performance scales linearly with $N$.
- **Strengths:** Produces soft shadows with penumbra; easy to integrate with existing ray tracing frameworks; user-controllable quality/performance trade-off [S1].
- **Limitations:** Requires area light sampling; may be noisy if $N$ is small; coherence benefits diminish if rays diverge significantly.
- **When to use:** When soft shadows are needed for area lights in participating media and interactive performance is required.
- **When not to use:** For point lights (hard shadows suffice) or when extremely high-quality shadows are needed without noise (requires large $N$, which is slow).
## Background
Originated in the context of coherent grid traversal for particle visualization [S1]. The approach leverages the fact that shadow rays from a single point to an area light share a common origin, allowing for efficient packet-based traversal. It is used in real-time or interactive visualization of complex particle datasets where global illumination effects like soft shadows are desired [S1].
## Sources
[S1] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
