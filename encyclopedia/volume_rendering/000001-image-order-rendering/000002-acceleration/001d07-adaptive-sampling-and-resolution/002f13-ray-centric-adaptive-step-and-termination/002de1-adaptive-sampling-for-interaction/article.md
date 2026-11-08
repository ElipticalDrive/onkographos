# Adaptive Sampling for Interaction

Also called: Adaptive Sampling for Interactive Volume Rendering, Dynamic Sampling Rate, Interaction-aware Adaptive Sampling, Two-rate Interaction Sampling

## Overview
Adaptive Sampling for Interaction is a performance optimization strategy that dynamically lowers the sampling rate during user interaction (e.g., camera movement or transfer function editing) to maintain interactive frame rates, then restores the high sampling rate once the interaction ceases [S1, S2]. Unlike [Adaptive Sampling](1385-adaptive-sampling.md), which adapts sampling density based on local data complexity to improve visual quality per sample, this method sacrifices visual fidelity temporarily to guarantee responsiveness. It builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by applying a global, time-dependent constraint on the sampling rate rather than a local, spatial one.

## How it works
The method operates in two distinct phases: an interaction phase with reduced sampling and a post-interaction phase with full resolution.

### Inputs and Outputs
*   **Inputs:** Scalar field $f(\mathbf{x})$, ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, transfer function $T(f)$, current interaction state (active/inactive), and timing parameters.
*   **Outputs:** Final pixel color $C$ and accumulated opacity $\alpha$.

### Data Structures
*   **Interaction State:** A boolean flag or timestamp indicating whether the user is currently interacting.
*   **Timing Buffer:** A record of the last interaction event time to determine when to switch back to high quality [S3, unofficial].

### Algorithm Steps

1.  **Interaction Detection:** Monitor user input events (mouse, keyboard). Set a flag `is_interacting` to `true` upon input and `false` after a hold period if no further input occurs [S3, unofficial].
2.  **Sampling Rate Selection:**
    *   If `is_interacting` is `true`, set the sampling rate $R_{int}$ to a low value (e.g., 1 sample per voxel or lower).
    *   If `is_interacting` is `false`, set the sampling rate $R_{high}$ to the target high value (e.g., 6–10 samples per voxel) [S1, S2].
3.  **Ray Marching:** Perform volume rendering using the selected sampling rate. The step size $\Delta t$ is determined by $1/R$.
    *   For interaction: $\Delta t_{int} = \frac{1}{R_{int}} \times \text{voxel\_size}$.
    *   For high quality: $\Delta t_{high} = \frac{1}{R_{high}} \times \text{voxel\_size}$.
4.  **Compositing:** Use standard front-to-back compositing with the calculated $\Delta t$ [S6].
    $$ \alpha_{seg} = 1 - e^{-f(\mathbf{x})\Delta t} $$
    $$ C_{seg} = T(f(\mathbf{x})) \cdot \alpha_{seg} $$
    $$ \alpha \leftarrow \alpha + \alpha_{seg}(1 - \alpha) $$
    $$ C \leftarrow C + C_{seg}(1 - \alpha) $$
5.  **Termination:** Terminate ray if $\alpha > 1 - \epsilon_{term}$ or ray exits volume [S6].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sampling_for_interaction(ray_origin, ray_direction, volume_data, transfer_func, params):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    volume_data: function sample(x) returning scalar f(x)
    transfer_func: function T(f) returning vec3 color
    params: dict containing:
        - high_rate: int (samples per voxel, e.g., 8)
        - low_rate: int (samples per voxel during interaction, e.g., 1)
        - is_interacting: bool (current interaction state)
        - hold_time_ms: int (milliseconds to wait after interaction stops before restoring high rate)
        - last_interaction_time: float (timestamp of last interaction)
        - current_time: float (current timestamp)
        - voxel_size: float (size of one voxel in world units)
        - opacity_threshold: float (1 - epsilon_term)
    """
    # Determine current sampling rate based on interaction state
    time_since_interaction = params['current_time'] - params['last_interaction_time']
    
    if params['is_interacting'] or time_since_interaction < params['hold_time_ms'] / 1000.0:
        current_rate = params['low_rate']
    else:
        current_rate = params['high_rate']
        
    # Calculate step size from rate
    # Assuming rate is samples per voxel, step size is voxel_size / rate
    step_size = params['voxel_size'] / current_rate
    
    t = 0.0
    accumulated_alpha = 0.0
    accumulated_color = vec3(0.0, 0.0, 0.0)
    
    # Determine entry and exit points
    t_start, t_end = compute_ray_bounds(ray_origin, ray_direction, volume_bounds)
    
    while t < t_end and accumulated_alpha < params['opacity_threshold']:
        pos = ray_origin + t * ray_direction
        f_val = volume_data.sample(pos)
        
        # Compositing
        alpha_seg = 1.0 - exp(-f_val * step_size)
        color_seg = transfer_func(f_val) * alpha_seg
        
        accumulated_alpha += alpha_seg * (1.0 - accumulated_alpha)
        accumulated_color += color_seg * (1.0 - accumulated_alpha)
        
        t += step_size
        
    return accumulated_color, accumulated_alpha
```

### Parameters and Typical Values
*   **High Sampling Rate:** 6 to 10 samples per voxel [S1, S2].
*   **Low Sampling Rate:** Not explicitly defined in [S1, S2], but implied to be significantly lower (e.g., 1 sample per voxel) to achieve interactivity.
*   **Hold Time:** 200 ms [S3, unofficial].
*   **Performance Impact:** At high sampling rates (6-10 samples/voxel), frame updates take nearly two seconds on an Octane2 and nearly one second on a GeForce3 [S1, S2]. Lowering the rate during interaction makes these times acceptable for interactivity.

### Edge Cases and Pitfalls
*   **Visual Coarseness:** During interaction, the rendered surface appears coarser. However, the surface size and location remain readily apparent, providing effective feedback for guiding user exploration [S1, S2].
*   **Transition Artifacts:** Switching between low and high rates may cause visible popping if not handled smoothly. A hold time (e.g., 200 ms) helps prevent flickering during rapid, small interactions [S3, unofficial].

## Subtopics

### Interaction Render Scale Damping
A variant that reduces the *render resolution* (image dimensions) rather than just the sampling rate along rays. It dynamically sets `interactionRenderScale` to 0.25 during camera movement and restores `userRenderScale` after a 200 ms hold period [S3, unofficial]. This reduces the number of rays cast, complementing the per-ray sampling reduction.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples. During interaction, the number of samples is reduced by a factor of 6–10, drastically reducing frame time from ~1-2 seconds to interactive rates (<33 ms).
*   **Memory Complexity:** No additional memory overhead compared to standard volume rendering, as it only changes the step size/rate.
*   **Strengths:** Guarantees interactive frame rates during user input; simple to implement; preserves visual fidelity when the user is not interacting.
*   **Limitations:** Visual quality is degraded during interaction; requires careful tuning of the hold time to avoid premature restoration of high quality or persistent low quality.
*   **When to Use:** Interactive volume rendering applications where user feedback speed is critical (e.g., medical imaging, scientific visualization).
*   **When Not to Use:** Offline rendering or applications where visual quality is paramount even during interaction.

## Background
This technique was introduced by Kindermann et al. in 2001 in the context of interactive volume rendering using multi-dimensional transfer functions [S1, S2]. The authors noted that high-quality rendering (6-10 samples per voxel) was too slow for interaction on hardware of the time (Octane2, GeForce3). By lowering the sample rate during interaction, they achieved effective feedback for transfer function exploration without sacrificing the final image quality once the user stopped interacting.

## Sources
[S1] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
[S3] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
