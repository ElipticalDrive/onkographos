# Soft Shadows via Ray Packets

Also called: Ray Packet Soft Shadows

## Overview
This method extends ray casting in participating media to support non-local illumination effects, specifically soft shadows, by tracing secondary ray packets from primary ray origins toward area light sources [S1]. It distinguishes itself from hard shadow techniques by sampling a spatial distribution of directions to approximate the penumbra, leveraging the coherence of these secondary rays to maintain interactive performance [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Primary ray packet (origin, direction, weight), scene geometry (particle system or mesh), light source definition (area and position).
*   **Output:** Shadow term (transmittance) applied to the primary ray's contribution.

**Data Structures**
The method operates on ray packets. A packet is a collection of $N$ rays sharing a common origin but diverging in direction to sample the light source area.
*   **Packet Size:** The number of shadow rays per primary ray is configurable. Tested sizes are $2 \times 2$ (4 rays), $4 \times 4$ (16 rays), and $8 \times 8$ (64 rays) [S1].
*   **Ray Layout:** Each ray consists of an origin point $\mathbf{o}$ and a direction vector $\mathbf{d}$ [G]. The origins of all rays in a shadow packet are identical to the primary ray's origin [G]. The directions are distributed across the solid angle subtended by the light source.

**Parameters**
*   **Light Source Area:** The size of the light source is defined as 5% of the solid angle subtended by the bounding box of the scene [S1].
*   **Packet Dimensions:** $k \times k$, where $k \in \{2, 4, 8\}$ [S1].

**Algorithm Steps**
1.  **Generate Shadow Directions:** For a given primary ray with origin $\mathbf{o}_p$, generate $N$ secondary ray directions $\{\mathbf{d}_i\}_{i=1}^{N}$ that uniformly or quasi-uniformly sample the solid angle of the light source [S1]. The area of the light source is fixed at 5% of the scene bounding box's solid angle [S1].
2.  **Construct Packet:** Create a ray packet where each ray $i$ has origin $\mathbf{o}_i = \mathbf{o}_p$ and direction $\mathbf{d}_i$ [S1].
3.  **Trace Rays:** Trace all $N$ rays in the packet against the scene geometry [S1].
    *   For each ray $i$, compute the transmittance $T_i$ from $\mathbf{o}_p$ to the light source. This involves checking for occlusions by particles or geometry along the path [G].
4.  **Aggregate Shadow Term:** Compute the final shadow term $S$ for the primary ray. A common approach for soft shadows is the average transmittance:
    $$ S = \frac{1}{N} \sum_{i=1}^{N} T_i $$
    [G] This assumes equal weighting for all samples in the packet.
5.  **Apply:** Multiply the primary ray's radiance contribution by $S$ [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_soft_shadow(primary_ray, scene, light_source, packet_size):
    """
    Computes soft shadow term for a primary ray using a ray packet.
    
    Args:
        primary_ray: Object with origin (o) and direction (d).
        scene: Object with method trace_ray(ray) returning transmittance.
        light_source: Object defining area and position.
        packet_size: Integer, dimension of the square packet (e.g., 2, 4, 8).
        
    Returns:
        float: Shadow term (0.0 to 1.0).
    """
    n = packet_size * packet_size
    total_transmittance = 0.0
    
    # Generate directions sampling the light source solid angle
    # The light source area is 5% of the solid angle of the scene bounding box [S1]
    directions = sample_light_source_directions(primary_ray.o, light_source, n)
    
    for d in directions:
        shadow_ray = Ray(origin=primary_ray.o, direction=d)
        # Trace ray against scene geometry/particles [S1]
        # Returns transmittance from origin to light source [G]
        t = scene.trace_ray(shadow_ray)
        total_transmittance += t
        
    return total_transmittance / n

def sample_light_source_directions(origin, light_source, count):
    """
    Generates 'count' directions from 'origin' that sample the light source.
    The light source subtends 5% of the solid angle of the scene bounding box [S1].
    """
    # Implementation depends on specific sampling strategy (e.g., stratified, uniform)
    # Not explicitly detailed in sources, but required for coherence [S1]
    # Placeholder for specific sampling algorithm [G]
    return [generate_direction(origin, light_source) for _ in range(count)]
```

**Edge Cases and Pitfalls**
*   **Coherence Dependency:** Interactive performance depends heavily on the coherence exhibited by secondary ray packets [S1]. If rays in a packet are not spatially or directionally coherent, acceleration structures (like grids) may not provide significant speedups, leading to poor performance [S1].
*   **Performance Degradation:** The performance impact is sublinear in the number of shadow rays, but large packets (e.g., $8 \times 8$) can significantly reduce frame rates [S1].
*   **Light Size:** The fixed light size (5% of bounding box solid angle) may not be appropriate for all scenes, potentially causing over-sampling or under-sampling if the scene scale varies significantly [S1].

## Operating characteristics
**Time and Memory Complexity**
*   **Time Complexity:** The cost is proportional to the number of shadow rays per primary ray ($N$) and the cost of tracing each ray. The impact is sublinear in $N$ due to coherence [S1].
*   **Memory Complexity:** Minimal additional memory for storing the packet of rays. The primary memory cost is the scene representation (particle grid or mesh) [S1].

**Measured Performance**
Performance was measured on various datasets using different packet sizes. Frame rates (fps) are provided below. The light source area was 5% of the solid angle subtended by the scene bounding box [S1].

| Dataset     | No Shadows | 2x2 Packets | 4x4 Packets | 8x8 Packets |
|-------------|------------|-------------|-------------|-------------|
| Cylinder    | 100.20     | 14.91       | 10.47       | 5.34        |
| JP8         | 99.88      | 16.31       | 12.21       | 6.65        |
| Bullet      | 126.93     | 19.55       | 13.69       | 6.56        |
| Thunder     | 40.86      | 14.07       | 10.74       | 6.45        |
| Foam        | 14.34      | 4.38        | 2.51        | 1.17        |
| BulletTorso | 18.30      | 7.39        | 6.09        | 3.82        |

[S1]

**Strengths and Limitations**
*   **Strengths:** Supports soft shadows with minimal code changes to existing ray casting pipelines [S1]. Performance degradation is sublinear, allowing for interactive frame rates with small packets (e.g., $2 \times 2$) [S1].
*   **Limitations:** Performance is highly dependent on ray coherence [S1]. Large packets ($8 \times 8$) can result in significant frame rate drops (up to 19.35x slower than no shadows) [S1]. The fixed light source size may not be optimal for all scenes [S1].

**When to Use**
*   Use when soft shadows are required and interactive performance is critical.
*   Use small packet sizes ($2 \times 2$) for high performance, larger sizes ($4 \times 4$ or $8 \times 8$) for higher quality if performance allows [S1].
*   Avoid when ray coherence is low (e.g., highly scattered media with no spatial structure) as performance gains from packet tracing may be negligible [S1].

## Background
This method was introduced in the context of interactive visualization of particle data, specifically within the "Coherent Grid Traversal" approach [S1]. It extends the basic ray casting technique to include non-local effects like shadows, which were previously omitted for performance reasons [S1]. The approach leverages the coherence of rays in particle datasets to maintain interactivity even when tracing multiple shadow rays per pixel [S1].

## Sources
[S1] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
