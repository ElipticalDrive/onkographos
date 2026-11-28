# Lag-First Traversal

Also called: Lag-first approach

## Overview
Lag-first traversal is a ray divergence handling technique used within Ray Packet (RP) marching to mitigate throughput degradation caused by rays entering distinct voxels [S1]. It selectively advances lagging rays within an RP to promote convergence with leading rays, thereby restoring ray-level parallelism and reducing redundant Empty Volume Traversal (EMA) operations [S1]. Unlike standard packet marching which may stall or process divergent rays inefficiently, this method actively synchronizes the group's spatial position [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A Ray Packet (RP) containing multiple rays that have diverged into distinct voxels during traversal [S1].
*   **Output:** An updated RP where lagging rays have been advanced to align with the leading rays, increasing the average number of simultaneous rays per RP [S1].

**Data Structures:**
*   **Ray Packet (RP):** A collection of rays processed in parallel. Each ray maintains its current position and traversal state [S1].
*   **Scheduler:** A logical unit that identifies divergence and manages ray advancement [S1].

**Steps:**
1.  **Divergence Detection:** The scheduler monitors rays within an RP. When rays enter distinct voxels, divergence is detected [S1].
2.  **Identification of Lagging Rays:** The scheduler identifies which rays are "lagging" behind the leading rays in the packet [S1].
3.  **Selective Advancement:** Lagging rays are selectively advanced in the traversal step to align with the group's leading position [S1]. This promotes convergence [S1].
4.  **Convergence:** The rays are brought back into proximity, allowing them to share traversal logic and memory accesses [S1].

**Parameters:**
*   **Average Simultaneous Rays:** The technique achieves an average of 3.89 simultaneous rays per RP during coarse traversal and 3.48 during fine traversal [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lag_first_traversal(ray_packet):
    """
    Advances lagging rays in a Ray Packet to promote convergence.
    
    Args:
        ray_packet: A list of Ray objects, each with a current voxel position.
        
    Returns:
        ray_packet: The updated Ray Packet with advanced lagging rays.
    """
    # Step 1: Identify the leading ray's voxel position
    # The leading ray is the one furthest along the traversal path
    leading_voxel = max(ray.ray_current_voxel for ray in ray_packet)
    
    # Step 2: Identify lagging rays
    # Rays not in the leading voxel are considered lagging
    lagging_rays = [ray for ray in ray_packet if ray.ray_current_voxel != leading_voxel]
    
    # Step 3: Selectively advance lagging rays
    for ray in lagging_rays:
        # Advance the ray to align with the group
        # The exact advancement logic depends on the traversal step size
        # and the distance to the leading voxel.
        # [G] Implementation detail: The ray is advanced by one or more steps
        # until it reaches the leading voxel or the next common voxel boundary.
        ray.advance_to(leading_voxel)
        
    return ray_packet
```

**Edge Cases and Pitfalls:**
*   **Precision:** Care must be taken to ensure that advancing lagging rays does not skip over valid data or cause overshooting, which could lead to incorrect sampling [S1].
*   **Boundaries:** At voxel boundaries, repeated loading and eviction of identical data can contribute to redundant EMAs if not managed by reordering RPs based on coarse voxel tags [S1].

## Subtopics

### Multiple Ray Tracing with Overtaking FIFO
A technique to hide the latency of the skip calculator by tracing multiple rays simultaneously, ensuring the pipeline remains full and avoiding stalls [S2]. It traces the path of more than one ray at a time (e.g., four rays) and keeps the current sample along each ray within close proximity to samples on other rays to ensure memory addresses are similar, minimizing page faults [S2]. It reads the voxel neighborhood (closest eight voxels) for a sample point from memory and trilinearly interpolates to calculate the new sample value [S2]. It introduces an Overtaking FIFO to reorder memory addresses to minimize page changes in memory and a Compositing Buffer to correctly sort samples for use in the Shading and Compositing stages [S2]. The VIZARD II architecture memory is eight times interleaved, allowing a 16 Msamples dataset read from 100MHz SDRAM to be scanned 50 times per second [S2]. The pipeline includes: Standard stages, Ray Queue, Skip Calculator, Address Calculation, SDRAM Control, Address FIFO, Voxel FIFO, Overtaking FIFO, and Compositing Buffer [S2]. A pitfall is that if one ray advances beyond others, reading voxels for the advanced ray causes page faults, stalling the ray casting during memory row activation and precharge time [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The technique achieves an average of 3.89 simultaneous rays per RP during coarse traversal and 3.48 during fine traversal [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Mitigates throughput degradation caused by ray divergence; promotes convergence of rays within an RP; reduces repetitive EMAs [S1].
*   **Limitations:** Requires additional scheduling logic to identify and advance lagging rays; may introduce complexity in managing ray states [S1].
*   **When to use:** When ray divergence within packets is significant and leads to reduced parallelism and increased redundant memory accesses [S1].
*   **When not to use:** When rays remain highly coherent and divergence is minimal, the overhead of the lag-first approach may not be justified [S1].

## Background
The lag-first approach was introduced as part of an energy-efficient edge coprocessor for neural rendering with explicit data reuse strategies [S1]. It addresses the critical challenge of ray divergence in RP-based marching, where rays within the same RP may enter distinct voxels, diminishing ray-level parallelism and increasing repetitive EMAs [S1]. The technique is employed in the second-stage scheduler of the described architecture [S1].

## Sources
[S1] An Energy-Efficient Edge Coprocessor for Neural Rendering With Explicit Data Reuse Strategies, https://doi.org/10.1109/tvlsi.2025.3641653, fetched 2026-10-26
[S2] PDF Efficient Space Leaping for Ray casting Architectures, https://fileadmin.cs.lth.se/cs/Personal/Michael_Doggett/pubs/volumeGraphcs01_SpaceLeaping.pdf, fetched 2026-10-28
