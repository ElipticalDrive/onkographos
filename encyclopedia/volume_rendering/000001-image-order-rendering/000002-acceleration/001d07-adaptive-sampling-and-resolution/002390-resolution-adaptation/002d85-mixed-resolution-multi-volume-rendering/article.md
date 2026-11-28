# Mixed-Resolution Multi-Volume Rendering

Also called: Per-channel resolution selection

## Overview
Mixed-resolution multi-volume rendering is a technique where different channels of a multi-volume dataset are rendered at different resolution levels to save memory and reduce sample evaluation [S1, unofficial][S2]. It addresses the inefficiency of applying a uniform high resolution to all channels when some possess lower frequency content or lower visual importance [S1, unofficial][S2]. It sets itself apart from sibling methods like [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md) by operating on a per-channel basis within a single ray traversal, rather than switching the entire dataset's resolution or using spatial partitioning for the whole volume [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A multi-volume dataset with $N$ channels, where each channel $c$ has a multi-resolution hierarchy [S1, unofficial][S2]. Viewing parameters including camera position and ray direction [S1, unofficial][S2].
*   **Output:** A rendered image where each pixel is composed of contributions from channels sampled at their individually selected resolution levels [S1, unofficial][S2].

**Data Structures:**
*   **Channel Resolution Map:** A function $R_c(p, t)$ for each channel $c \in \{0, \dots, N-1\}$ and spatial position $p$ along ray parameter $t$ [S1, unofficial][S2]. This determines the resolution level $l$ to use for channel $c$ at position $p$ [S1, unofficial][S2].
*   **Multi-resolution Hierarchy:** Each channel $c$ stores data at discrete resolution levels $l$ [S1, unofficial][S2]. Level $l=0$ is the highest resolution level (finest grid), and higher $l$ values represent coarser resolutions [S1, unofficial][S2].

**Algorithm Steps:**
1.  **Initialize Ray:** For each pixel, cast a ray through the volume [G].
2.  **Traverse Hierarchy:** Iterate through the spatial structure (e.g., octree or brick hierarchy) along the ray [S1, unofficial][S2].
3.  **Determine Resolution per Channel:** For the current sample point $p$ and ray parameter $t$, evaluate the resolution constraint function for each channel $c$:
    $$ l_c = \text{clamp}(R_c(p, t), l_{\min}, l_{\max}) $$
    The function $R_c$ constrains the range of valid resolution levels [S1, unofficial][S2]. It may consider distance to the camera, frequency content of the channel, user-defined bounds, and transfer function importance [S1, unofficial][S2].
4.  **Select Data:** For each channel $c$, retrieve the voxel value from its hierarchy at level $l_c$ [S1, unofficial][S2].
5.  **Sample:** Evaluate the channel value at the sample point using the data from level $l_c$ [S1, unofficial][S2]. Note that if channels have different resolutions, the sampling rate along the ray may effectively differ per channel [S2]. A high-frequency channel might be sampled at every step, while a low-frequency channel might only need sampling every second step [S1, unofficial][S2].
6.  **Composite:** Accumulate the color and opacity contributions from all channels using standard volume rendering integration (e.g., front-to-back compositing) [G].
7.  **Repeat:** Continue until the ray exits the volume or a termination condition is met [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mixed_resolution_ray(ray_origin, ray_direction, volume_bounds, channel_data, channel_configs):
    # volume_bounds: tuple of (min_point, max_point), each point is (x,y,z)
    # channel_data: dict mapping channel_id -> MultiResolutionHierarchy object
    # channel_configs: dict mapping channel_id -> function(pos, t, origin, dir) -> resolution_level
    color = [0.0, 0.0, 0.0]
    alpha = 0.0
    
    # Compute ray-volume intersection interval [G]
    t_min = 0.0
    t_max = float('inf')
    for i in range(3):
        if abs(ray_direction[i]) > 1e-6:
            t1 = (volume_bounds[0][i] - ray_origin[i]) / ray_direction[i]
            t2 = (volume_bounds[1][i] - ray_origin[i]) / ray_direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
        else:
            if ray_origin[i] < volume_bounds[0][i] or ray_origin[i] > volume_bounds[1][i]:
                return color
                
    if t_min > t_max:
        return color
        
    dt = 0.5  # Base step size [G]
    t = t_min
    
    while t < t_max:
        pos = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
        
        channel_samples = {}
        for cid, hierarchy in channel_data.items():
            # Evaluate resolution constraint function for channel cid [S1, unofficial][S2]
            level = channel_configs[cid](pos, t, ray_origin, ray_direction)
            # Retrieve and interpolate value from hierarchy at selected level [S1, unofficial][S2]
            value = hierarchy.sample(level, pos)
            channel_samples[cid] = value
            
        # Composite channels using front-to-back blending [G]
        for cid, value in channel_samples.items():
            # Map channel value to color and opacity via transfer function [G]
            c_color = [value, value, value]  # Standard transfer function mapping [G]
            c_alpha = min(value, 1.0)
            
            weight = c_alpha * (1.0 - alpha)
            color[0] += c_color[0] * weight
            color[1] += c_color[1] * weight
            color[2] += c_color[2] * weight
            alpha += weight
            if alpha >= 1.0:
                break
                
        t += dt
        
    return color
```

**Parameters:**
*   **Resolution Constraint Function:** The specific logic for $R_c$. Can be user-defined bounds or a complex function [S1, unofficial][S2]. Typical example values from human tissue data: pink channel at $l_3$, orange channel at $l_5/l_4/l_0$, and rest at $l_0$ [S1, unofficial].
*   **Resolution Levels:** Discrete levels $l_0, \dots, l_{max}$. $l_0$ is the highest resolution level [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Aliasing:** A channel with high-frequency content may require a higher sampling rate (lower $l$) to avoid aliasing, while another may only need sampling at every second step [S1, unofficial][S2].
*   **Memory Management:** Limiting the range of resolutions uploaded to the GPU on a per-channel basis saves memory [S1, unofficial][S2].
*   **Synchronization:** Ensuring that sampling positions for different channels are consistent or appropriately interpolated if their grid spacings differ significantly [G].

## Subtopics

### Frequency-Based Resolution Selection
Chooses resolution based on the spectral content of each channel. High-frequency channels use higher resolution [S1, unofficial][S2]. Sets itself apart by analyzing channel-specific frequency spectra rather than spatial distance [G]. Choose when channels have distinct frequency characteristics [G]. [Link to parent](3063-resolution-adaptation.md).

### View-Dependent Resolution Selection
Chooses resolution based on the distance to the camera or viewing angle. Distant objects/channels use lower resolution [S1, unofficial][S2]. Sets itself apart by using camera-relative metrics instead of intrinsic data properties [G]. Choose when viewpoint significantly affects perceived detail [G]. [Link to parent](3063-resolution-adaptation.md).

### Importance-Based Resolution Selection
Chooses resolution based on the visual importance of the channel, often derived from the transfer function. Less important channels use lower resolution [S1, unofficial][S2]. Sets itself apart by weighting channels against user-defined visual priorities [G]. Choose when some channels are secondary or supplementary [G]. [Link to parent](3063-resolution-adaptation.md).

## Operating characteristics
*   **Time Complexity:** Depends on the number of channels and the average resolution level selected. If $N$ channels are rendered, and the average number of samples per ray is $S$, the complexity is roughly $O(N \cdot S)$ [G]. However, by reducing the resolution for less important channels, $S$ can be reduced for those channels, leading to an overall speedup [S1, unofficial][S2].
*   **Memory Complexity:** Reduced by only uploading the necessary resolution levels for each channel to the GPU. The memory footprint is proportional to the sum of the sizes of the selected resolution levels for all channels [S1, unofficial][S2].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Significant memory savings for large multi-volume datasets. Reduced computational cost for less important or low-frequency channels [S1, unofficial][S2].
*   **Limitations:** Potential for visual artifacts if resolution selection is not well-tuned. Complexity in managing different sampling rates for different channels [S1, unofficial][S2].
*   **When to Use:** Large-scale multi-volume datasets where channels have varying frequency content or importance. Out-of-core rendering scenarios [S1, unofficial][S2].
*   **When Not to Use:** Single-channel volumes or when all channels have similar frequency content and importance [G].

## Background
This technique is described in the context of scalable web-based volume rendering of large biomedical datasets [S1, unofficial][S2]. It is part of a system that uses out-of-core methods and multi-resolution hierarchies (e.g., OME-Zarr files) to handle large data volumes [S1, unofficial][S2]. The approach allows for flexible resolution selection per channel based on various factors, enabling efficient rendering of complex multi-channel data [S1, unofficial][S2].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
