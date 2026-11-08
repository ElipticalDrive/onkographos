# Mixed-Resolution Multi-Volume Rendering
Also called: Per-channel resolution selection

## Overview
Mixed-resolution multi-volume rendering is a technique where different channels of a multi-volume dataset are rendered at different spatial resolution levels to save memory and reduce sample evaluation costs [S1, unofficial][S2]. It addresses the inefficiency of rendering all channels at the highest resolution when some channels have lower frequency content or lower visual importance [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A multi-volume dataset with $N$ channels, where each channel $i$ has a multi-resolution hierarchy (e.g., a brick hierarchy or octree) with resolution levels $l_0$ (highest) to $l_{max}$ (lowest) [S1, unofficial][S2]. Viewing parameters such as camera position and ray direction [S1, unofficial][S2].
*   **Output:** A rendered image where each pixel's color is computed by integrating contributions from all channels, sampled at their respective selected resolution levels [S1, unofficial][S2].

**Data Structures:**
*   **Channel Resolution Constraint Function:** For each channel $i$, a function $f_i(\mathbf{p}, \mathbf{v}, \dots)$ that returns a valid resolution level $l_i \in [l_{min}, l_{max}]$ [S1, unofficial][S2]. The inputs $\mathbf{p}$ (position), $\mathbf{v}$ (viewing parameters), and potentially frequency content or transfer function properties [S1, unofficial][S2].
*   **Multi-resolution Hierarchy:** Each channel's data is stored in a hierarchy of resolution levels [S1, unofficial][S2]. Level $l_0$ is the finest, and higher $l$ values represent coarser resolutions [S1, unofficial][S2].

**Steps:**
1.  **Initialize Ray:** Set ray origin $\mathbf{t}$, step size $\Delta t$, and accumulation buffers for color and opacity [G].
2.  **Determine Resolution per Channel:** For each channel $i$, evaluate the constraint function $f_i$ using the current ray sample position and viewing parameters to determine the resolution level $l_i$ to use for that channel [S1, unofficial][S2].
3.  **Sample Data:** For each channel $i$, sample the volume data at the determined resolution level $l_i$ [S1, unofficial][S2].
4.  **Integrate:** Compute the volume rendering integral for each channel using the sampled data at their respective resolutions, then blend contributions into the final pixel buffer [S1, unofficial][S2].
5.  **Advance Ray:** Update $\mathbf{t} \leftarrow \mathbf{t} + \mathbf{d} \cdot \Delta t$ and repeat until the ray exits the volume bounds or opacity reaches a threshold [G].
6.  **Memory Management:** Limit the range of resolutions uploaded to the GPU on a per-channel basis to save memory and reduce evaluated samples [S1, unofficial][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_mixed_resolution_multi_volume(ray_origin, ray_direction, channels, camera, volume_bounds):
    """
    ray_origin: list of 3 floats [x, y, z]
    ray_direction: list of 3 floats [dx, dy, dz]
    channels: list of channel objects, each with .constraint_function(pos, camera) and .sample_at(pos, level)
    camera: object with position and view parameters
    volume_bounds: tuple of (min_point, max_point) defining the axis-aligned bounding box
    """
    final_color = [0.0, 0.0, 0.0, 0.0]
    t = ray_origin[:]
    dt = 0.01  # [G] Typical initial step size based on bounding box or highest resolution
    max_steps = 1000  # [G] Safety limit to prevent infinite loops

    for _ in range(max_steps):
        if not is_inside_volume(t, volume_bounds):
            break

        # Determine resolution for each channel at current position
        channel_resolutions = []
        for channel in channels:
            l_i = channel.constraint_function(t, camera)
            channel_resolutions.append(l_i)

        # Sample and integrate each channel at its selected resolution
        for i, channel in enumerate(channels):
            l_i = channel_resolutions[i]
            density, color = channel.sample_at(t, l_i)

            alpha = 1.0 - math.exp(-density * dt)
            final_color[0] += color[0] * alpha * (1.0 - final_color[3])
            final_color[1] += color[1] * alpha * (1.0 - final_color[3])
            final_color[2] += color[2] * alpha * (1.0 - final_color[3])
            final_color[3] += alpha * (1.0 - final_color[3])

            if final_color[3] >= 0.99:
                break

        # Advance ray
        t[0] += ray_direction[0] * dt
        t[1] += ray_direction[1] * dt
        t[2] += ray_direction[2] * dt

    return final_color

def is_inside_volume(point, bounds):
    """[G] Standard bounding box check."""
    min_pt, max_pt = bounds
    return all(min_pt[j] <= point[j] <= max_pt[j] for j in range(3))
```

**Parameters:**
*   **Resolution Levels:** Discrete levels $l_0, \dots, l_{max}$ where $l_0$ is highest resolution [S1, unofficial][S2].
*   **Constraint Function Inputs:** Distance to camera, frequency content, and transfer function properties [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Frequency Mismatch:** A channel with high-frequency content may require a higher sampling rate, while another may only need sampling at every second step [S1, unofficial][S2].
*   **Memory Management:** Limiting the range of resolutions uploaded to the GPU is crucial for out-of-core rendering [S1, unofficial][S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of channels and the selected resolution levels [S1, unofficial][S2]. Rendering at lower resolutions for some channels reduces the number of samples evaluated [S1, unofficial][S2].
*   **Memory Complexity:** Reduces memory usage by limiting the range of resolutions uploaded to the GPU on a per-channel basis [S1, unofficial][S2].
*   **Strengths:** Saves memory and reduces computational cost for multi-volume datasets where channels have varying importance or frequency content [S1, unofficial][S2].
*   **Limitations:** Requires a multi-resolution hierarchy for each channel and a mechanism to determine the appropriate resolution level per channel [S1, unofficial][S2].
*   **When to Use:** Large-scale multi-volume data where some channels have lower frequency content or are less important for the user [S1, unofficial][S2].

## Background
Mixed-resolution multi-volume rendering was introduced in the context of scalable web-based volume rendering of large biomedicine datasets [S1, unofficial][S2]. It is part of a system that uses out-of-core methods and multi-resolution hierarchies (e.g., OME-Zarr files) to handle large data volumes [S2]. The technique allows for flexible resolution selection per channel based on viewing parameters and data characteristics [S1, unofficial][S2].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
