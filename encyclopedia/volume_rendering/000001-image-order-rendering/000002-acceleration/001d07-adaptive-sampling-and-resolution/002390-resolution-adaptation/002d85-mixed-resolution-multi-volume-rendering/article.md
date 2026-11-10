# Mixed-Resolution Multi-Volume Rendering

Also called: Per-channel resolution selection

## Overview
Mixed-resolution multi-volume rendering is a technique where different channels of a multi-volume dataset are rendered at different resolution levels to save memory and reduce sample evaluation [S1, unofficial][S2]. It addresses the inefficiency of applying a uniform high resolution to all channels when some possess lower frequency content or lower visual importance [S1, unofficial][S2]. It sets itself apart from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by operating on a per-channel basis within a single ray traversal, rather than switching the entire dataset's resolution or using spatial partitioning for the whole volume.

## How it works
**Inputs and Outputs:**
*   **Input:** A multi-volume dataset with $N$ channels, where each channel $c$ has a multi-resolution hierarchy (e.g., levels $l_0$ to $l_{max}$). Viewing parameters (camera position, ray direction).
*   **Output:** A rendered image where each pixel is composed of contributions from channels sampled at their individually selected resolution levels.

**Data Structures:**
*   **Channel Resolution Map:** A mapping or function $R_c(p)$ for each channel $c \in \{0, \dots, N-1\}$ and spatial position $p$. This determines the resolution level $l$ to use for channel $c$ at position $p$.
*   **Multi-resolution Hierarchy:** Each channel $c$ stores data at discrete resolution levels $l$. Level $l=0$ is the highest resolution (finest grid), and higher $l$ values represent coarser resolutions.

**Algorithm Steps:**
1.  **Initialize Ray:** For each pixel, cast a ray through the volume.
2.  **Traverse Hierarchy:** Iterate through the spatial structure (e.g., octree or brick hierarchy) along the ray.
3.  **Determine Resolution per Channel:** For the current spatial position $p$ (or sample point) and current ray parameter $t$:
    *   Evaluate the resolution constraint function for each channel $c$:
        $$ l_c = R_c(p, t, \text{viewing\_params}, \text{frequency\_content}_c, \text{transfer\_function}_c) $$
    *   The function $R_c$ constrains the range of valid resolution levels. It may consider:
        *   Distance to the camera.
        *   Frequency content of the channel (high-frequency channels may require lower $l$, i.e., higher resolution).
        *   User-defined bounds.
        *   Transfer function importance.
4.  **Select Data:** For each channel $c$, retrieve the voxel value from its hierarchy at level $l_c$.
5.  **Sample:** Evaluate the channel value at the sample point using the data from level $l_c$. Note that if channels have different resolutions, the sampling rate along the ray may effectively differ per channel. A high-frequency channel might be sampled at every step, while a low-frequency channel might only need sampling every second step [S1, unofficial][S2].
6.  **Composite:** Accumulate the color and opacity contributions from all channels using standard volume rendering integration (e.g., front-to-back or back-to-front compositing).
7.  **Repeat:** Continue until the ray exits the volume or a termination condition is met.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mixed_resolution_ray(ray_origin, ray_direction, volume_data, channel_configs):
    """
    ray_origin: vec3
    ray_direction: vec3
    volume_data: dict mapping channel_id -> MultiResolutionHierarchy
    channel_configs: dict mapping channel_id -> ResolutionConstraintFunction
    """
    color = vec3(0.0)
    alpha = 0.0
    
    # Standard ray marching setup
    t = 0.0
    dt = calculate_step_size(ray_direction, volume_data) # Base step size, may vary per channel
    
    while t < ray_length:
        pos = ray_origin + t * ray_direction
        
        # Determine resolution for each channel at this position
        channel_samples = {}
        for channel_id, hierarchy in volume_data.items():
            # Get resolution level for this channel at current position
            # The function can depend on viewing params, frequency, etc.
            level = channel_configs[channel_id](pos, t, ray_origin, ray_direction)
            
            # Sample the channel at the selected resolution level
            # Note: Different channels may have different effective sampling rates
            # if their resolution levels imply different grid spacings.
            # For simplicity, we assume we sample at the current t for all,
            # but the data comes from different levels.
            value = hierarchy.sample(level, pos)
            channel_samples[channel_id] = value
            
        # Composite all channels
        # This step depends on the specific compositing model (e.g., alpha blending)
        # and how multi-channel data (e.g., RGB, RGBA, or separate scalars) is handled.
        # Assuming a simple alpha blending model for demonstration:
        for channel_id, value in channel_samples.items():
            # Get color and opacity for this channel value
            c_color, c_alpha = get_color_and_opacity(channel_id, value)
            
            # Front-to-back compositing
            weight = c_alpha * (1.0 - alpha)
            color += c_color * weight
            alpha += weight
            
            if alpha >= 1.0:
                break
                
        # Advance ray
        # The step size dt might be adjusted per channel if their resolutions differ significantly
        # to avoid aliasing or unnecessary samples.
        t += dt
        
    return color
```

**Parameters:**
*   **Resolution Constraint Function:** The specific logic for $R_c$. Can be user-defined bounds or a complex function. Typical values are not given; it is problem-dependent [S1, unofficial][S2].
*   **Resolution Levels:** Discrete levels $l_0, \dots, l_{max}$. $l_0$ is finest [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Aliasing:** A channel with high-frequency content may require a higher sampling rate (lower $l$) to avoid aliasing, while another may only need sampling at every second step [S1, unofficial][S2].
*   **Memory Management:** Limiting the range of resolutions uploaded to the GPU on a per-channel basis saves memory [S1, unofficial][S2].
*   **Synchronization:** Ensuring that the sampling positions for different channels are consistent or appropriately interpolated if their grid spacings differ significantly.

## Subtopics

### Frequency-Based Resolution Selection
Chooses resolution based on the spectral content of each channel. High-frequency channels use higher resolution. [Link to specific sub-article if exists, otherwise link to parent].

### View-Dependent Resolution Selection
Chooses resolution based on the distance to the camera or viewing angle. Distant objects/channels use lower resolution. [Link to specific sub-article if exists, otherwise link to parent].

### Importance-Based Resolution Selection
Chooses resolution based on the visual importance of the channel, often derived from the transfer function. Less important channels use lower resolution. [Link to specific sub-article if exists, otherwise link to parent].

## Operating characteristics
*   **Time Complexity:** Depends on the number of channels and the average resolution level selected. If $N$ channels are rendered, and the average number of samples per ray is $S$, the complexity is roughly $O(N \cdot S)$. However, by reducing the resolution for less important channels, $S$ can be reduced for those channels, leading to an overall speedup.
*   **Memory Complexity:** Reduced by only uploading the necessary resolution levels for each channel to the GPU. The memory footprint is proportional to the sum of the sizes of the selected resolution levels for all channels.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Significant memory savings for large multi-volume datasets. Reduced computational cost for less important or low-frequency channels.
*   **Limitations:** Potential for visual artifacts if resolution selection is not well-tuned. Complexity in managing different sampling rates for different channels.
*   **When to Use:** Large-scale multi-volume datasets where channels have varying frequency content or importance. Out-of-core rendering scenarios.
*   **When Not to Use:** Single-channel volumes or when all channels have similar frequency content and importance.

## Background
This technique is described in the context of scalable web-based volume rendering of large biomedical datasets [S1, unofficial][S2]. It is part of a system that uses out-of-core methods and multi-resolution hierarchies (e.g., OME-Zarr files) to handle large data volumes. The approach allows for flexible resolution selection per channel based on various factors, enabling efficient rendering of complex multi-channel data.

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
