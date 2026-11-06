# Dynamic Frame Rate Adaptation

Also called: Automatic resolution adoption, Dynamic resolution distribution

## Overview
Dynamic Frame Rate Adaptation is a runtime adjustment system that tracks rendering performance and adjusts the distribution of resolution levels on the fly to maintain a consistently high update rate above a set threshold [S1]. It solves the problem of performance instability in volume rendering, where small parametric changes (e.g., transfer function modifications) can cause frame rate dips leading to cybersickness [S1]. It sets itself apart from static resolution schemes by continuously shifting the "anchor points" between resolution levels based on the gradient of recent frame times, rather than using fixed blending factors [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sliding window of rendering times for the last $n$ frames; current frame rendering time; lower and upper refresh rate thresholds; the current Multi-Anchor Blending (MAB) configuration.
*   **Output:** Updated blending factors (resolution level distribution) for the next frame.

**Data Structures:**
*   **Performance Window:** A circular buffer or array storing the rendering time $t_i$ for the last $n$ frames.
*   **Lookup Texture ($f_a$):** A 1D texture used as a fast lookup function. It maps texel coordinates to resolution levels.
    *   **Layout:** The texture has a width corresponding to the domain of the blending function.
    *   **Channels:** The four color channels (RGBA) store the blending factors for the corresponding resolution levels of the MAB [S1].
    *   **Content:** Each channel value represents the weight of a specific resolution level. A smooth-step function is applied between adjacent resolution levels to ensure unnoticeable blending [S1].

**Parameters:**
*   $n$: Size of the sliding window (number of past frames). Not given in the sources.
*   $T_{low}$: Lower threshold for minimum acceptable refresh rate (e.g., 90 Hz) [S1].
*   $T_{high}$: Upper threshold for sufficiently high refresh rate. Not given in the sources.
*   $\Delta t$: Change in rendering time over the last $n$ frames.

**Steps:**
1.  **Track Performance:** Maintain a sliding window of the last $n$ frame rendering times $[t_{-n}, ..., t_{-1}]$ [S1].
2.  **Compare Thresholds:** Compare the previous frame's rendering time $t_{-1}$ against the lower threshold $T_{low}$ and upper threshold $T_{high}$ [S1].
    *   If the frame rate is below $T_{low}$ (performance is poor), the system must shift the resolution distribution to emit fewer rays (favor performance) [S1].
    *   If the frame rate is above $T_{high}$ (performance is good), the system increases the number of rays (favor quality) [S1].
3.  **Compute Gradient:** Calculate the gradient of the rendering time window to determine the direction and magnitude of the adjustment. This involves evaluating the change in rendering time over the last $n$ frames [S1].
4.  **Select Anchor Point:** Determine which anchor point between two adjacent resolution levels should be moved based on the computed gradient [S1].
5.  **Determine Shift Magnitude:** Calculate how far the selected anchor point should be shifted in the corresponding direction (towards lower or higher resolution) [S1].
6.  **Update Lookup Texture:** Modify the $f_a$ 1D texture. Adjust the values in the RGBA channels to reflect the new blending factors for each resolution level. This effectively changes the "sizes" of each resolution level in the domain [S1].
7.  **Raycasting:** During the raycast, use the updated $f_a$ texture to look up the corresponding resolution level for each texel and determine if it needs to be traversed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_resolution_distribution(performance_window, current_frame_time, t_low, t_high, n):
    """
    Adjusts the MAB resolution distribution based on frame rate performance.
    
    Args:
        performance_window: List of rendering times for the last n frames.
        current_frame_time: Rendering time of the most recent frame.
        t_low: Lower threshold for acceptable refresh rate (in seconds).
        t_high: Upper threshold for high refresh rate (in seconds).
        n: Number of frames in the window.
        
    Returns:
        new_blending_factors: Array of blending factors for resolution levels.
    """
    # Append current frame time to window
    performance_window.append(current_frame_time)
    if len(performance_window) > n:
        performance_window.pop(0)
        
    # Calculate gradient of rendering time over the window
    # Gradient indicates if performance is improving or degrading
    # Simple implementation: difference between last and first in window
    time_change = performance_window[-1] - performance_window[0]
    
    # Determine target direction based on current frame rate
    # Note: Thresholds are in Hz in sources, but time is in seconds.
    # Conversion: Time threshold = 1 / Frequency threshold.
    # Assuming t_low and t_high are pre-converted to seconds for this logic.
    
    if current_frame_time > t_low:
        # Performance is too slow (frame rate below threshold)
        # Shift distribution to fewer rays (lower resolution)
        direction = -1 
    elif current_frame_time < t_high:
        # Performance is too fast (frame rate above threshold)
        # Increase rays (higher resolution)
        direction = 1
    else:
        # Within acceptable range, no major shift needed
        direction = 0
        
    # Determine which anchor point to move and by how much
    # This depends on the specific MAB implementation details not fully provided
    # We assume a function that maps direction and gradient to anchor shift
    anchor_index, shift_magnitude = select_anchor_and_shift(performance_window, direction, time_change)
    
    # Update the blending factors (fa texture)
    # The fa texture maps domain coordinates to resolution levels
    # We adjust the anchor point between resolution levels
    blending_factors = update_fa_texture(anchor_index, shift_magnitude)
    
    return blending_factors

def select_anchor_and_shift(window, direction, gradient):
    """
    Determines which anchor point to move and the magnitude of the shift.
    Minor variations affect higher resolution levels; high deviations affect lower resolutions.
    """
    # Placeholder for specific logic not detailed in sources
    # Sources state: "evaluating the change in rendering time... determines which anchor point... 
    # and how far it should be shifted"
    # And: "Minor variations lead to adjustments in higher resolution levels... 
    # high deviations primarily shift lower resolutions"
    
    # Implementation note:
    # If |gradient| is small, adjust anchor near higher resolution levels.
    # If |gradient| is large, adjust anchor near lower resolution levels.
    
    # For the purpose of a complete implementation, we assume a linear mapping
    # from gradient magnitude to anchor index and shift amount.
    
    # Normalize gradient to a 0-1 scale relative to expected variance
    max_expected_gradient = 0.1 # Example value, not in sources
    normalized_gradient = min(abs(gradient) / max_expected_gradient, 1.0)
    
    # Map normalized gradient to anchor index (0 = lowest res, N = highest res)
    # Higher gradient -> lower resolution anchor
    num_anchors = 4 # Example, depends on MAB setup
    anchor_index = int(normalized_gradient * (num_anchors - 1))
    
    # Shift magnitude is proportional to gradient
    shift_magnitude = gradient * 0.1 # Scaling factor, not in sources
    
    return anchor_index, shift_magnitude
```

**Edge Cases and Pitfalls:**
*   **Minor Variations:** Small performance fluctuations primarily adjust higher resolution levels, causing only slight alterations to the overall distribution [S1].
*   **High Deviations:** Large performance deviations primarily shift lower resolutions to compensate for rendering performance [S1].
*   **Precision:** The smooth-step function between adjacent resolution levels is critical for unnoticeable blending; discontinuities here may cause visual artifacts [S1].

## Operating characteristics
*   **Time Complexity:** The adjustment logic involves a sliding window update and a lookup texture modification, which is $O(1)$ per frame relative to the scene complexity, though dependent on $n$ for gradient calculation. The raycasting cost is reduced by shifting to lower resolutions when performance is poor.
*   **Memory Complexity:** Requires storage for the performance window ($O(n)$) and the $f_a$ lookup texture (constant size, typically small 1D texture) [S1].
*   **Measured Performance:** On an RTX 2070, the baseline system maintained average refresh rates above 90 Hz for MRI and CT datasets. With dynamic adaptation, it maintained rates above 90 Hz on both RTX 2070 and RTX 2060m hardware, whereas the baseline failed on the RTX 2060m for one dataset [S1]. The 1% lowest frame rates also improved (e.g., MRI dataset on RTX 2060m: 62.6 Hz baseline vs 91.2 Hz dynamic) [S1].
*   **Strengths:** Guarantees refresh rates above a set threshold, reducing cybersickness. Adapts to runtime parametric changes (e.g., transfer function updates) [S1].
*   **Limitations:** Relies on accurate tracking of frame times. The specific mapping of gradient to anchor shift is not fully detailed, potentially requiring tuning [S1].

## Background
The method originates from research on embracing raycasting for virtual reality, specifically addressing the need for consistent high update rates to prevent cybersickness [S1]. It builds on the Multi-Anchor Blending (MAB) technique, using a 1D lookup texture ($f_a$) similar to transfer functions in direct volume rendering to manage resolution level blending [S1]. The system was tested on MRI and CT datasets using RTX 2070 and RTX 2060m hardware [S1].

## Sources
[S1] Embracing Raycasting for Virtual Reality, https://doi.org/10.24132/csrn.3201.16, fetched 2026-10-29
