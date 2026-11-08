# Adaptive Sampling for Interaction

Also called: Adaptive Sampling for Interactive Volume Rendering, Dynamic Sampling Rate, Interaction-aware Adaptive Sampling, Two-rate Interaction Sampling

## Overview
Adaptive Sampling for Interaction is a performance optimization strategy that dynamically reduces the volume sampling rate during user interaction (such as camera movement or transfer function adjustment) to maintain interactive frame rates, then restores the high sampling rate once the interaction ceases [S1, S2]. It differs from static resolution scaling by being context-aware and time-bound, specifically decoupling performance from user input latency [S1, S2]. Unlike [Adaptive Sampling](1385-adaptive-sampling.md), which adapts to data complexity, this method adapts to user state. It builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by applying temporal adaptation to the sampling density rather than just step size.

## How it works
The method operates in two distinct phases: an interaction phase with reduced sampling and a post-interaction phase with full sampling.

### Inputs and Outputs
*   **Inputs:** Scalar field $f(\mathbf{x})$, ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, transfer function $T(f)$, current user interaction state (active/inactive), and sampling parameters.
*   **Outputs:** Final pixel color $C$ and accumulated opacity $\alpha$.

### Data Structures
*   **Interaction State:** A boolean or timestamp flag indicating whether the user is currently interacting with the view or parameters [S1, S2].
*   **Sampling Configuration:** Two sets of sampling parameters:
    *   **High Rate:** Used for final rendering or static views.
    *   **Low Rate:** Used during interaction.

### Algorithm Steps
1.  **Detect Interaction State:** Determine if the user is currently modifying the camera or transfer function [S1, S2].
2.  **Select Sampling Rate:**
    *   If interaction is active, select the **Low Rate** [S1, S2].
    *   If interaction is inactive (or completed), select the **High Rate** [S1, S2].
3.  **Ray Marching:** Perform volume rendering using the selected sampling rate. The underlying ray marching loop follows the principles of [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md), but the base step size $\Delta t$ is determined by the global interaction state rather than local data metrics [S1, S2].
4.  **Transition:** When interaction ceases, the system switches back to the High Rate for subsequent frames [S1, S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_with_interaction_adaptation(ray_origin, ray_direction, volume_data, transfer_func, is_interacting):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    volume_data: function sample(x) returning scalar f(x)
    transfer_func: function T(f) returning vec3 color
    is_interacting: bool, True if user is currently interacting
    
    Returns:
    accumulated_color: vec3
    accumulated_alpha: float
    """
    # Define sampling rates
    # High rate: 6-10 samples per voxel [S1, S2]
    # Low rate: Not explicitly defined, but implied to be significantly lower [S1, S2]
    # We define a ratio or specific step sizes. 
    # Let's assume base_step_high corresponds to 1/6 to 1/10 voxel size.
    
    if is_interacting:
        # Use lower sampling rate during interaction [S1, S2]
        # Exact factor not specified, but must be low enough for interactivity [S1, S2]
        samples_per_voxel = 1.0 # Placeholder for "low rate", e.g., 1-2 samples/voxel
    else:
        # Use high sampling rate after interaction [S1, S2]
        samples_per_voxel = 8.0 # Typical value from [S1, S2] (range 6-10)
    
    # Calculate step size based on samples per voxel
    # Assuming voxel size is 1.0 for simplicity, or derived from volume bounds
    voxel_size = 1.0 
    step_size = voxel_size / samples_per_voxel
    
    t = 0.0
    accumulated_alpha = 0.0
    accumulated_color = vec3(0.0, 0.0, 0.0)
    
    # Determine ray bounds (simplified)
    t_start, t_end = compute_ray_bounds(ray_origin, ray_direction, volume_bounds)
    
    while t < t_end and accumulated_alpha < 0.99:
        pos = ray_origin + t * ray_direction
        f_val = volume_data.sample(pos)
        
        # Compositing
        alpha_seg = 1.0 - exp(-f_val * step_size)
        color_seg = transfer_func(f_val) * alpha_seg
        
        accumulated_alpha += alpha_seg * (1.0 - accumulated_alpha)
        accumulated_color += color_seg * (1.0 - accumulated_alpha)
        
        t += step_size
        
        if accumulated_alpha >= 0.99:
            break
            
    return accumulated_color, accumulated_alpha
```

### Parameters and Typical Values
*   **High Sampling Rate:** 6 to 10 samples per voxel [S1, S2].
*   **Low Sampling Rate:** Not explicitly given in the sources, but must be sufficient to maintain "effective feedback" [S1, S2].
*   **Interaction Threshold:** Not explicitly defined; typically based on user input events [S1, S2].

### Edge Cases and Pitfalls
*   **Visual Quality Drop:** During interaction, the volume rendered surface appears coarser [S1, S2]. This is an accepted trade-off for performance [S1, S2].
*   **Feedback Adequacy:** The coarser surface must still allow the user to perceive surface size and location for effective transfer function exploration [S1, S2].

## Subtopics

### [Interaction Render Scale Damping](8891-interaction-render-scale-damping.md)
Interaction Render Scale Damping is a strategy within adaptive sampling that dynamically reduces the render resolution during user interaction (such as camera movement) to maintain a target frame rate, then restores the original resolution after interaction ceases. It differs from static resolution scaling by being time-bound and context-aware, specifically designed to decouple performance from user input latency. It builds on [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md).

## Operating characteristics
*   **Performance:** At high sampling rates (6-10 samples/voxel), frame updates take nearly two seconds on an Octane2 and nearly one second on a GeForce3 [S1, S2]. Reducing the sampling rate during interaction significantly improves frame rates, enabling interactivity [S1, S2].
*   **Visual Quality:** The rendered surface appears coarser during interaction, but surface size and location remain readily apparent, providing effective feedback for guiding user exploration [S1, S2].
*   **Strengths:** Maintains interactivity during complex operations; simple to implement as a global state switch.
*   **Limitations:** Visual quality fluctuates; requires careful tuning of the low sampling rate to ensure feedback remains useful.

## Background
This technique was introduced in the context of Interactive Volume Rendering Using Multi-Dimensional Transfer Functions [S1, S2]. The authors noted that high-quality rendering (6-10 samples/voxel) was too slow for interactive exploration on hardware available at the time (Octane2, GeForce3) [S1, S2]. To address this, they proposed lowering the sample rate during interaction and re-rendering at the higher rate once the action was completed [S1, S2].

## Sources
[S1] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
