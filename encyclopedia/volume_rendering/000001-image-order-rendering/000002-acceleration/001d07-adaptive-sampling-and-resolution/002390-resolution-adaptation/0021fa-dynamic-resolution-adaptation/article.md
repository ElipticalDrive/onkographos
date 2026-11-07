# Dynamic Resolution Adaptation

Also called: ComputeReductionFactor, Frame Rate Maintenance via Resolution Adjustment

## Overview
Dynamic Resolution Adaptation is a runtime performance management technique that adjusts the render viewport reduction factor based on previous frame render time and an allocated time budget to maintain a target frame rate [S1, unofficial]. It solves the problem of performance instability in volume rendering, where small parametric changes (e.g., transfer function modifications) can cause frame rate dips leading to cybersickness [S1, unofficial]. It sets itself apart from static resolution schemes by continuously shifting the "anchor points" between resolution levels based on the gradient of recent frame times, rather than using fixed blending factors [S1, unofficial].

## How it works
The method operates as a feedback loop that predicts rendering time and dynamically selects the level-of-detail (LOD) or resolution for the next frame [S2, unofficial].

**Inputs and Outputs:**
*   **Inputs:** Previous frame render time ($T_{draw}$), allocated time budget for the next frame ($T_{budget}$), current reduction factor ($R_{current}$), and desired frame rate parameters [S1, unofficial][S2, unofficial].
*   **Outputs:** New reduction factor ($R_{new}$) or selected LOD index for the next frame [S1, unofficial][S2, unofficial].

**Data Structures:**
*   **Reduction Factor:** A scalar value $R \in (0.0, 1.0]$ representing the viewport scaling [S1, unofficial].
*   **Performance Statistics:** A history of frame times used to predict rendering time [S2, unofficial].

**Steps:**
1.  **Metric Evaluation:** Collect performance statistics to predict the rendering time for the next frame [S2, unofficial].
2.  **Resolution Adjustment:** Compute a new reduction factor $R_{new}$ ensuring it stays within the valid range $(0.0, 1.0]$ [S1, unofficial]. The adjustment is based on the difference between the allocated time budget and the predicted/actual render time [S1, unofficial].
3.  **Rendering/Integration:** Render the scene using the adjusted resolution or selected LOD [S2, unofficial].
4.  **Stability/Coherence:** The system tracks rendering performance to maintain a consistently high update rate above a set threshold [S1, unofficial].

**Formulas:**
Let $T_{draw}$ be the time taken to draw the previous frame [S1, unofficial].
Let $T_{budget}$ be the allocated time in seconds for the next frame [S1, unofficial].
Let $R_{current}$ be the current reduction factor [S1, unofficial].

The new reduction factor $R_{new}$ is computed such that:
$$ 0.0 < R_{new} \le 1.0 $$
[S1, unofficial]

The specific mapping function from $T_{draw}$ and $T_{budget}$ to $R_{new}$ is not explicitly defined in the sources, but the logic implies an inverse relationship: if $T_{draw} > T_{budget}$, $R_{new}$ should decrease; if $T_{draw} < T_{budget}$, $R_{new}$ may increase [S1, unofficial].

**Parameters:**
*   **allocatedTime ($T_{budget}$):** A double in seconds representing the time budget for the next frame [S1, unofficial].
*   **Desired frame rate:** A parameter used to match the level-of-detail selection [S2, unofficial].

**Edge Cases and Pitfalls:**
*   The algorithm requires the existing $R_{current}$ to be $> 0.0$ and $\le 1.0$ [S1, unofficial].
*   The algorithm requires $T_{draw} \ge 0.0$ [S1, unofficial].
*   Potential artifacts, like discontinuities, can occur when using mixed resolutions; these can be avoided by using different proxy geometries other than planes or by modifying transfer functions in lower-resolution nodes [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_resolution_adaptation(T_draw, T_budget, R_current):
    """
    Computes the new reduction factor for the next frame.
    
    Args:
        T_draw (float): Time taken to draw the previous frame in seconds.
        T_budget (float): Allocated time budget for the next frame in seconds.
        R_current (float): Current reduction factor, must be in (0.0, 1.0].
        
    Returns:
        float: New reduction factor R_new in (0.0, 1.0].
    """
    # Guard clauses based on pitfalls
    if R_current <= 0.0 or R_current > 1.0:
        raise ValueError("Reduction factor must be in (0.0, 1.0]")
    if T_draw < 0.0:
        raise ValueError("Render time cannot be negative")
    if T_budget <= 0.0:
        raise ValueError("Time budget must be positive")

    # The specific formula for R_new is not given in the sources [S1, unofficial].
    # The method ensures R_new stays in range (0.0, 1.0] based on T_draw and T_budget [S1, unofficial].
    # Implementation detail: A common approach is to scale R based on the ratio of T_budget to T_draw.
    # However, since the exact formula is not provided, we return the current R as a placeholder
    # or implement a simple proportional adjustment if assumed.
    # [G] Assuming a simple proportional adjustment for implementation completeness:
    # If T_draw > T_budget, we need to reduce resolution (lower R).
    # If T_draw < T_budget, we can increase resolution (higher R).
    
    # Calculate a target ratio
    if T_draw == 0:
        ratio = 1.0
    else:
        ratio = T_budget / T_draw
        
    # Adjust R_current based on ratio. 
    # Note: The exact mapping logic is not specified in [S1, unofficial].
    # This is a generic implementation of the concept.
    R_new = R_current * ratio
    
    # Clamp to valid range
    if R_new > 1.0:
        R_new = 1.0
    elif R_new <= 0.0:
        # Avoid zero or negative, use a small epsilon or minimum valid value
        R_new = 1e-6 # [G] Small positive value to prevent division by zero or invalid state
        
    return R_new
```

## Subtopics

### [Dynamic Frame Rate Adaptation](8221-dynamic-frame-rate-adaptation.md)
Dynamic Frame Rate Adaptation is a runtime adjustment system that tracks rendering performance and adjusts the distribution of resolution levels on the fly to maintain a consistently high update rate above a set threshold. It solves the problem of performance instability in volume rendering, where small parametric changes (e.g., transfer function modifications) can cause frame rate dips leading to cybersickness. It sets itself apart from static resolution schemes by continuously shifting the "anchor points" between resolution levels based on the gradient of recent frame times, rather than using fixed blending factors.

## Operating characteristics
*   **Time Complexity:** The adaptation logic itself is $O(1)$ per frame, involving simple arithmetic operations on performance metrics [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** Requires storage for performance statistics (history of frame times) and the current reduction factor [S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The method scales with the complexity of the volume rendering, as the prediction of rendering time relies on historical performance data [S2, unofficial].
*   **Strengths:** Maintains a consistent frame rate, preventing cybersickness in interactive applications [S1, unofficial].
*   **Limitations:** Potential visual artifacts (discontinuities) when switching resolutions [S2, unofficial]. Requires careful tuning of the time budget and reduction factor clamping [S1, unofficial].
*   **When to use:** Interactive volume rendering where frame rate stability is critical [S1, unofficial].
*   **When not to use:** Offline rendering where visual quality is prioritized over frame rate [S2, unofficial].

## Background
The concept of automatic level-of-detail selection to match desired frame rates by predicting rendering time and dynamically selecting the level-of-detail for the next frame is known as time-critical level-of-detail selection [S2, unofficial]. This approach was developed by Li et al. [S2, unofficial]. The specific implementation using a `ComputeReductionFactor` based on `allocatedTime` and `TimeToDraw` is found in the VTKMultiVolumeRayCaster project [S1, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
