# Mixed-Resolution Multi-Volume Renderer


## Overview
A ray-guided renderer that accesses metadata and cached volume data through the [Residency Octree](4950-residency-octree.md) to enable efficient mixing of resolutions across different channels in multi-volume datasets [S1, unofficial]. It is implemented as a WebGPU compute shader that evaluates each visible channel at every sample along a ray, even when channels have different resolutions, to reduce shader divergence [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Ray origin, ray direction, current node index in the full octree, channel mapping, maximum visible channels $m$, and user-defined parameters [S1, unofficial].
- **Output:** Computed color and opacity for the ray segment, updated ray position, and potential data streaming requests [S1, unofficial].

**Data Structures:**
- **Residency Octree:** Provides metadata (for empty-space skipping) and cached volume data (bricks) [S1, unofficial].
- **Full Octree Layout:** Nodes are accessed directly via their index without root traversal [S1, unofficial].
- **Channel Mapping:** Maps dataset channels to the $m$ visible channels ($m \le n$, where $n$ is total channels); this mapping can change at runtime [S1, unofficial].

**Parameters:**
- $m$: Maximum visible channels, user-defined at initialization [S1, unofficial].
- $stepScale$: User-defined scaling factor to increase or decrease the sampling rate at run-time [S1, unofficial].
- Channel Importance: Determined by user-defined resolution ranges via UI sliders; a channel with a higher upper bound in its resolution range is considered more important [S1, unofficial].

**Step Size Calculation:**
The sampling rate is determined globally for all channels based on the channel requiring the highest sampling rate (highest resolution) [S1, unofficial]. The step size $dt_{step}$ is computed as follows:
1. Calculate the step size component for each dimension $i \in [0, 1, 2]$:
   $$ dt_i = \frac{1}{v_i \cdot s_i \cdot d_i} $$
   Where $v_i$ is the volume extent in dimension $i$, $s_i$ is the voxel spacing in dimension $i$, and $d_i$ is the ray direction in dimension $i$ [S1, unofficial].
2. Compute the global step size:
   $$ dt_{step} = \min(dt_0, dt_1, dt_2) \cdot stepScale $$

**Algorithm Steps:**
1. **Initialization:** Set the maximum visible channels $m$ and channel mapping [S1, unofficial].
2. **Traversal Start:** Start tree traversal at a user-defined subdivision level or the parent level of the previous termination level; never start at the root [S1, unofficial].
3. **Data Check:** For the current node, check if metadata and cached volume data are available [S1, unofficial].
   - If missing, generate a request for the missing data to be streamed from the server [S1, unofficial].
4. **Empty-Space Skipping:** Use metadata to skip empty nodes [S1, unofficial].
5. **Sampling:** For each sample along the ray:
   - Compute the global step size $dt_{step}$ based on the highest-resolution visible channel [S1, unofficial].
   - Evaluate all $m$ visible channels at the current sample position to reduce diverging branches in the shader [S1, unofficial].
   - Accumulate color and opacity.
6. **Termination:** Stop when the ray exits the volume or opacity threshold is reached.
7. **Loop:** Repeat for subsequent rays or frames [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, current_node_index, channel_mapping, step_scale, max_res_v, max_res_s):
    # Compute global step size based on highest-resolution channel [S1, unofficial]
    dt_components = []
    for i in range(3):
        if ray_direction[i] == 0.0:
            dt_components.append(float('inf'))  # Guard for zero direction component [G]
        else:
            dt_i = 1.0 / (max_res_v[i] * max_res_s[i] * ray_direction[i])
            dt_components.append(dt_i)
    dt_step = min(dt_components) * step_scale
    
    node_idx = current_node_index
    color = vec3(0.0)
    opacity = 0.0
    
    while node_idx is not None:
        # Check data availability [S1, unofficial]
        if not has_metadata(node_idx) or not has_cached_data(node_idx):
            request_streaming(node_idx)  # Comment: Sources do not specify streaming protocol [G]
            # Proceed if data is available or wait for async completion [G]
            
        # Empty-space skipping [S1, unofficial]
        if is_empty(node_idx):
            node_idx = get_next_node(node_idx)  # Comment: Standard octree traversal logic [G]
            continue
            
        # Sample along ray segment covered by this node [S1, unofficial]
        # Evaluate all visible channels at each sample to reduce divergence [S1, unofficial]
        while opacity < 1.0:
            sample_pos = ray_origin + ray_direction * dt_step
            for ch in channel_mapping:
                val = sample_volume(ch, sample_pos)  # Comment: Standard trilinear interpolation [G]
                tf_color, tf_alpha = transfer_function(ch, val)  # Comment: Standard transfer function lookup [G]
                color += tf_color * tf_alpha * (1.0 - opacity)
                opacity += tf_alpha * (1.0 - opacity)
            ray_origin = sample_pos
            if opacity >= 1.0:
                break
            
        node_idx = get_next_node(node_idx)  # Comment: Advance to next node in traversal order [G]
        
    return color
```

**Edge Cases and Pitfalls:**
- **Ray Direction Zero:** If a ray direction component is zero, $dt_i$ becomes infinite. Handle this by taking the minimum of non-infinite components or using a large finite value [G].
- **Missing Data:** Streaming requests may stall rendering. Implement asynchronous handling or fallbacks [G].
- **Channel Mapping Changes:** Ensure smooth transitions when channel mapping changes at runtime [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of samples per ray, which is determined by $dt_{step}$. Evaluating all $m$ channels at each sample adds a factor of $m$ [G].
- **Memory Complexity:** Depends on the number of cached bricks in the residency octree [G].
- **Performance:** Reduces shader divergence by evaluating all channels at each sample. Global step size simplifies sampling logic [S1, unofficial].
- **Strengths:** Efficient handling of multi-resolution data, reduced divergence, ray-guided data streaming [S1, unofficial].
- **Limitations:** Potential over-sampling if lower-resolution channels are evaluated at high resolution. Requires careful tuning of $stepScale$ and channel importance [S1, unofficial].

## Background
The mixed-resolution multi-volume renderer is part of a system for scalable web-based volume rendering of large biomedical datasets [S1, unofficial]. It leverages the residency octree for data management and is implemented as a WebGPU compute shader [S1, unofficial]. The CPU drives overall rendering, dispatches rendering and memory management commands to the GPU, forwards brick requests to the server, and uploads bricks received from the server to the GPU [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
