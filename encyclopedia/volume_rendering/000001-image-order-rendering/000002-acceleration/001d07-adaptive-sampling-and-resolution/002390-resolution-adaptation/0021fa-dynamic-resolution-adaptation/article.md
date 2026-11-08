# Dynamic Resolution Adaptation

Also called: ComputeReductionFactor, Frame Rate Maintenance via Resolution Adjustment

## Overview
Dynamic Resolution Adaptation is a performance-based acceleration technique that dynamically adjusts the render viewport reduction factor based on previous frame render time and allocated time budget to maintain a target frame rate [S1, unofficial]. It solves the problem of performance instability in volume rendering, where small parametric changes (e.g., transfer function modifications) can cause frame rate dips leading to cybersickness [S1, unofficial]. It sets itself apart from static resolution schemes by continuously shifting the "anchor points" between resolution levels based on the gradient of recent frame times, rather than using fixed blending factors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `ReductionFactor` (current viewport scale, $n \in (0.0, 1.0]$), `TimeToDraw` (actual time taken to render the previous frame, $t_{actual} \ge 0.0$), `allocatedTime` (target time budget for the next frame, $t_{target}$).
*   **Output:** New `ReductionFactor` for the current frame.

**Data Structures:**
No complex data structures are required. The method relies on scalar values for timing and resolution scaling.

**Steps:**
1.  **Validate State:** Ensure the current `ReductionFactor` is in the valid range $(0.0, 1.0]$ and `TimeToDraw` is non-negative [S1, unofficial]. If invalid, the method cannot proceed correctly [S1, unofficial].
2.  **Compute Error:** Calculate the difference between the allocated time budget and the actual time taken:
    $$ \Delta t = t_{target} - t_{actual} $$
3.  **Adjust Reduction Factor:** Update the reduction factor based on the time error. The goal is to reduce resolution (lower $n$) if the frame took too long, or increase resolution (raise $n$) if there was spare time. The exact update rule typically involves a proportional adjustment or a step function to ensure stability, ensuring the new factor remains within $(0.0, 1.0]$ [S1, unofficial].
4.  **Apply Scaling:** Use the new `ReductionFactor` to scale the viewport dimensions for the current frame's rendering pass.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_reduction_factor(current_reduction_factor, time_to_draw, allocated_time):
    """
    Adjusts the viewport reduction factor to maintain target frame rate.
    
    Args:
        current_reduction_factor (float): Current n in (0.0, 1.0].
        time_to_draw (float): Actual render time of previous frame in seconds.
        allocated_time (float): Target time budget for next frame in seconds.
        
    Returns:
        float: New reduction factor in (0.0, 1.0].
    """
    # Guard clauses based on pitfalls [S1, unofficial]
    if current_reduction_factor <= 0.0 or current_reduction_factor > 1.0:
        raise ValueError("ReductionFactor must be in (0.0, 1.0]")
    if time_to_draw < 0.0:
        raise ValueError("TimeToDraw must be >= 0.0")

    # Calculate time error
    time_error = allocated_time - time_to_draw
    
    # Adjust reduction factor
    # If time_error > 0, we have spare time, so we can increase resolution (move towards 1.0)
    # If time_error < 0, we are over budget, so we must decrease resolution (move towards 0.0)
    
    # Simple proportional adjustment with clamping
    # Note: The exact coefficient for adjustment is not specified in the source [S1, unofficial]
    # A common implementation uses a smoothing factor or fixed step.
    # Here we assume a simple linear adjustment towards the ideal factor of 1.0 scaled by error.
    
    # To prevent oscillation, a smoothing factor or hysteresis is often applied [S1, unofficial]
    # Since the source does not specify the exact formula, we implement a basic clamp-based update.
    
    # Estimate ideal factor based on time ratio
    # If we rendered in half the time, we could theoretically render at 2x resolution (capped at 1.0)
    # If we took double the time, we need half resolution.
    if time_to_draw > 0:
        ideal_factor = current_reduction_factor * (allocated_time / time_to_draw)
    else:
        ideal_factor = 1.0 # No time taken, assume max resolution possible
        
    # Clamp to valid range
    new_reduction_factor = max(0.001, min(1.0, ideal_factor))
    
    return new_reduction_factor
```

**Parameters:**
*   `allocatedTime`: Double in seconds representing the time budget for the next frame [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The method requires an existing `ReductionFactor` $> 0.0$ and $\le 1.0$, and `TimeToDraw` $\ge 0.0$ [S1, unofficial].
*   Without smoothing or hysteresis, the resolution may oscillate rapidly between frames, causing visual artifacts and performance instability [S1, unofficial].

## Subtopics

### [Dynamic Frame Rate Adaptation](8221-dynamic-frame-rate-adaptation.md)
Dynamic Frame Rate Adaptation is a runtime adjustment system that tracks rendering performance and adjusts the distribution of resolution levels on the fly to maintain a consistently high update rate above a set threshold. It solves the problem of performance instability in volume rendering, where small parametric changes (e.g., transfer function modifications) can cause frame rate dips leading to cybersickness. It sets itself apart from static resolution schemes by continuously shifting the "anchor points" between resolution levels based on the gradient of recent frame times, rather than using fixed blending factors.

### Time-critical level-of-detail selection
An algorithm that performs automatic level-of-detail selection to match different parameters like the desired frame rate by predicting rendering time and dynamically selecting the level-of-detail for the next frame [S2, unofficial]. It attempts to predict the rendering time by collecting performance statistics and dynamically selecting the level-of-detail for the next frame [S2, unofficial]. The user can change parameters so regions of interest are rendered with higher quality [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for the adaptation logic itself. The rendering cost scales with the square of the reduction factor (for 2D viewport scaling) or linearly with volume samples if step size is adjusted.
*   **Memory Complexity:** $O(1)$ additional memory for state variables.
*   **Strengths:** Maintains a consistent frame rate, preventing cybersickness in interactive applications. Simple to implement.
*   **Limitations:** Can cause visible resolution changes between frames if not smoothed. Does not account for visual complexity directly, only performance metrics.
*   **When to use:** Interactive volume rendering where frame rate stability is critical.
*   **When not to use:** Offline rendering where visual quality is prioritized over frame rate, or when visual error metrics are available and preferred.

## Background
The concept of dynamically adjusting resolution based on performance metrics originates from the need to maintain interactivity in volume rendering. The specific implementation of "ComputeReductionFactor" is found in the VTKMultiVolumeRayCaster project [S1, unofficial]. The broader concept of time-critical level-of-detail selection was developed by Li et al. [S2, unofficial] to match rendering parameters to desired frame rates by predicting rendering time.

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
