# Dynamic Resolution Adaptation

Also called: ComputeReductionFactor, Frame Rate Maintenance via Resolution Adjustment

## Overview
Dynamic Resolution Adaptation is a runtime performance management technique that adjusts the render viewport reduction factor based on previous frame render time and an allocated time budget to maintain a target frame rate [S1, unofficial]. It solves the problem of performance instability in volume rendering, where small parametric changes (e.g., transfer function modifications) can cause frame rate dips leading to cybersickness [S1, unofficial]. It sets itself apart from static resolution schemes by continuously shifting the "anchor points" between resolution levels based on the gradient of recent frame times, rather than using fixed blending factors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `TimeToDraw`: The actual time taken to render the previous frame [S1, unofficial].
    *   `allocatedTime`: The time budget in seconds for the next frame [S1, unofficial].
    *   `ReductionFactor`: The current viewport reduction factor, constrained to $(0.0, 1.0]$ [S1, unofficial].
*   **Outputs:**
    *   `NewReductionFactor`: The updated viewport reduction factor for the current frame [S1, unofficial].

**Data Structures:**
No complex data structures are required. The method relies on scalar values for timing and resolution scaling.

**Steps:**
1.  **Validate State:** Ensure the current `ReductionFactor` is strictly greater than $0.0$ and less than or equal to $1.0$, and that `TimeToDraw` is non-negative [S1, unofficial]. If these conditions are not met, the adaptation logic cannot proceed safely [S1, unofficial].
2.  **Compute Metric:** Calculate the ratio of the actual render time to the allocated time budget. This ratio indicates whether the renderer is behind or ahead of schedule [S1, unofficial].
3.  **Adjust Resolution:** Update the `ReductionFactor` based on the metric. The goal is to reduce the resolution (increase the reduction factor) if the frame time exceeded the budget, or increase the resolution (decrease the reduction factor) if there was spare time [S1, unofficial].
4.  **Clamp:** Ensure the new `ReductionFactor` remains within the valid range $(0.0, 1.0]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_reduction_factor(current_reduction_factor, time_to_draw, allocated_time):
    """
    Adjusts the viewport reduction factor to maintain target frame rate.
    
    Args:
        current_reduction_factor (float): Current reduction factor in (0.0, 1.0].
        time_to_draw (float): Time taken to render the previous frame in seconds.
        allocated_time (float): Target time budget for the next frame in seconds.
        
    Returns:
        float: The new reduction factor.
    """
    # Pitfall: Requires existing ReductionFactor to be >0.0 and <=1.0, 
    # and TimeToDraw >= 0.0 [S1, unofficial]
    if current_reduction_factor <= 0.0 or current_reduction_factor > 1.0:
        raise ValueError("ReductionFactor must be in (0.0, 1.0]")
    if time_to_draw < 0.0:
        raise ValueError("TimeToDraw must be >= 0.0")
    if allocated_time <= 0.0:
        raise ValueError("AllocatedTime must be > 0.0")

    # Compute the performance ratio
    # If time_to_draw > allocated_time, we are slow and need to reduce resolution
    # If time_to_draw < allocated_time, we are fast and can increase resolution
    # The exact formula for the new factor is not explicitly detailed in the source,
    # but the principle is to map the metric to the resolution parameter [S1, unofficial].
    # A common implementation uses a proportional or exponential smoothing factor.
    
    # Placeholder for the specific adjustment formula not provided in [S1, unofficial]
    # The source states: "Compute new ReductionFactor ensuring it stays in range (0.0, 1.0]"
    # We assume a linear or proportional adjustment based on the error.
    
    error = time_to_draw - allocated_time
    
    # [G] Generic smoothing factor to prevent oscillation. 
    # The source does not specify the exact gain or smoothing constant.
    smoothing_factor = 0.1 
    
    # Adjust reduction factor:
    # Positive error (slow) -> increase reduction factor (lower res)
    # Negative error (fast) -> decrease reduction factor (higher res)
    new_reduction_factor = current_reduction_factor + (error / allocated_time) * smoothing_factor
    
    # Clamp to valid range (0.0, 1.0]
    # Note: 0.0 is excluded to prevent division by zero or infinite resolution
    if new_reduction_factor <= 0.0:
        new_reduction_factor = 0.001 # [G] Small epsilon to stay > 0.0
    elif new_reduction_factor > 1.0:
        new_reduction_factor = 1.0
        
    return new_reduction_factor
```

**Parameters:**
*   `allocatedTime`: Double in seconds representing the time budget for the next frame [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The method requires the existing `ReductionFactor` to be $>0.0$ and $\le 1.0$, and `TimeToDraw` $\ge 0.0$ [S1, unofficial].
*   If the frame time is significantly higher than the budget, the reduction factor may hit the upper bound ($1.0$), potentially resulting in unacceptable visual quality.
*   If the frame time is significantly lower, the reduction factor may hit the lower bound (near $0.0$), which may be computationally expensive or unstable.

## Subtopics

### Time-critical level-of-detail selection
Time-critical level-of-detail selection is an algorithm that performs automatic level-of-detail selection to match different parameters like the desired frame rate by predicting rendering time and dynamically selecting the level-of-detail for the next frame [S2, unofficial]. It sets itself apart from Dynamic Resolution Adaptation by operating on discrete LOD levels rather than a continuous viewport reduction factor, and by using performance statistics to *predict* future rendering time rather than reacting to the immediate previous frame's time [S2, unofficial]. It is chosen when discrete LOD structures (like octrees) are available and prediction-based selection is preferred over reactive adjustment [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The adaptation logic itself is $O(1)$ in time and memory, as it involves simple arithmetic operations on scalar values [S1, unofficial]. The rendering time complexity depends on the chosen reduction factor and the underlying volume rendering algorithm.

**Measured Performance:**
Not given in the sources.

**Scaling:**
The method scales with the complexity of the volume rendering pipeline. As the volume size or transfer function complexity increases, the `TimeToDraw` will increase, triggering more aggressive resolution reduction [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Maintains a consistent frame rate, reducing cybersickness in interactive applications [S1, unofficial]. Simple to implement [S1, unofficial].
*   **Limitations:** Can lead to visual popping if the resolution changes are too abrupt. Requires careful tuning of the adjustment parameters to avoid oscillation [S1, unofficial].

**When to Use:**
Use when maintaining a stable frame rate is critical, such as in virtual reality or real-time interactive volume visualization [S1, unofficial].

**When Not to Use:**
Do not use when visual fidelity is more important than frame rate stability, or when the rendering pipeline cannot support dynamic resolution changes [S1, unofficial].

## Background
Dynamic Resolution Adaptation is implemented in the VTKMultiVolumeRayCaster project [S1, unofficial]. It is designed to handle multiple volumes and maintain performance stability [S1, unofficial]. The concept of time-critical level-of-detail selection is discussed in the context of volume visualization on large-scale displays [S2, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
