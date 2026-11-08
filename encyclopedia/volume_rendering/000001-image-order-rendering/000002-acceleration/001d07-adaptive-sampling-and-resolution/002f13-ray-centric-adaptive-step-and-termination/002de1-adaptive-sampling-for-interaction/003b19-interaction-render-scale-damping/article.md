# Interaction Render Scale Damping

Also called: Dynamic Resolution Damping, Interaction-aware Resolution Scaling

## Overview
Interaction Render Scale Damping is a strategy within adaptive sampling that dynamically reduces the render resolution during user interaction (such as camera movement) to maintain a target frame rate, then restores the original resolution after interaction ceases [S1, unofficial]. It differs from static resolution scaling by being time-bound and context-aware, specifically designed to decouple performance from user input latency [S1, unofficial]. It builds on [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md).

## How it works
**Inputs and Outputs:**
*   **Input:** Current user interaction state (active/inactive), `userRenderScale` (the intended full-quality scale), and `interactionRenderScale` (the reduced scale).
*   **Output:** A render resolution multiplier applied to the frame buffer and ray casting grid.

**Data Structures:**
*   `interactionRenderScale`: A float in the range $[0, 1]$. A value of 0 disables scaling (full resolution) [S1, unofficial].
*   `userRenderScale`: A float representing the user's desired resolution scale when idle [S1, unofficial].
*   `lastMoveTime`: A timestamp recording the most recent user interaction event [S1, unofficial].
*   `INTERACTION_HOLD_MS`: A constant delay (typically 200ms) before restoring full resolution [S1, unofficial].

**Steps:**
1.  **Detect Interaction:** Monitor user input events (e.g., mouse drag, keyboard navigation).
2.  **Update Timestamp:** On any interaction event, update `lastMoveTime` to the current time $t_{now}$ [S1, unofficial].
3.  **Apply Damping:** If interaction is active, set the active render scale to `interactionRenderScale` [S1, unofficial].
4.  **Check Restoration:** If interaction is inactive, calculate the elapsed time $\Delta t = t_{now} - lastMoveTime$.
5.  **Restore Scale:** If $\Delta t > INTERACTION_HOLD\_MS$, transition the active render scale back to `userRenderScale` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Configuration
INTERACTION_HOLD_MS = 200
interactionRenderScale = 0.25
userRenderScale = 1.0
lastMoveTime = 0
isInteracting = False

def on_interaction_event():
    global lastMoveTime, isInteracting
    isInteracting = True
    lastMoveTime = current_time_ms()

def on_interaction_end():
    global isInteracting
    isInteracting = False

def get_active_render_scale():
    global isInteracting, lastMoveTime, userRenderScale, interactionRenderScale
    
    if isInteracting:
        return interactionRenderScale
    
    elapsed = current_time_ms() - lastMoveTime
    if elapsed > INTERACTION_HOLD_MS:
        return userRenderScale
    else:
        # Still in hold period, keep reduced resolution
        return interactionRenderScale
```

**Parameters:**
*   `interactionRenderScale`: Default 0.25 [S1, unofficial]. This reduces linear dimensions by half (area by 4x) if derived from ray count reduction, or simply scales the buffer size.
*   `INTERACTION_HOLD_MS`: Default 200ms [S1, unofficial]. Prevents flickering between resolutions during rapid, short interactions.

**Edge Cases:**
*   **Precision:** Ensure `current_time_ms()` has sufficient resolution to handle the 200ms threshold accurately [S1, unofficial].
*   **Degenerate Input:** If `interactionRenderScale` is 0, the system should fall back to `userRenderScale` or handle the division by zero if used as a divisor [S1, unofficial].

## Subtopics

### Reduced Resolution Ray Casting
An interactive rendering technique that trades image quality for speed by reducing the number of samples or image resolution [S2, book]. It generates a reduced resolution image and upsamples it to full size using nearest neighbor or bilinear interpolation [S2, book]. If bilinear interpolation is used, the number of rays cast is often reduced by a factor of two along each image dimension during interaction, resulting in a four-times speed-up without a noticeable decrease in image quality [S2, book]. Further speed-ups with larger reductions result in blurry, less detailed images [S2, book]. It may also implement progressive refinement, computing only every $n^{th}$ ray during interaction and filling in the rest with interpolation, which are then progressively replaced with actual values when interaction stops [S2, book].

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of pixels and samples. Reducing resolution by a factor of 2 in each dimension reduces pixel count by 4x, yielding a 4x speed-up [S2, book].
*   **Memory Complexity:** Lower resolution requires less frame buffer memory and potentially fewer ray samples stored in temporary buffers [S2, book].
*   **Strengths:** Maintains high frame rates during interaction, improving perceived responsiveness [S1, unofficial].
*   **Limitations:** Image quality degrades during interaction; aggressive scaling leads to blurriness [S2, book].
*   **When to Use:** Interactive volume exploration where frame rate is critical [S1, unofficial].
*   **When Not to Use:** Final rendering or static analysis where image fidelity is paramount [S2, book].

## Background
Originating from techniques described in advanced computer graphics literature for volume rendering, this method addresses the need for interactive rates in image-order ray casting [S2, book]. It is implemented in modern web-native volume renderers like Kiln to handle multi-GB datasets efficiently [S1, unofficial].

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
