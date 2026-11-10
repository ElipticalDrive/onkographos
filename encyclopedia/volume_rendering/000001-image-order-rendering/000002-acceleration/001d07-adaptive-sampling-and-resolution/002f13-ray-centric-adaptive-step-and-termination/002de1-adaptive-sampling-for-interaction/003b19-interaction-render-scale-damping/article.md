# Interaction Render Scale Damping

Also called: Dynamic Resolution Damping, Interaction-aware Resolution Scaling

## Overview
Interaction Render Scale Damping is a performance optimization strategy that dynamically reduces the rendering resolution during user interaction (such as camera movement) to maintain a high frame rate, then restores the target resolution after interaction ceases [S1, unofficial]. It differs from static resolution scaling by being time-triggered and context-aware, specifically designed to bridge the gap between interactive responsiveness and final image quality in volume rendering pipelines.

## How it works
The method operates by overriding the current render scale with a lower value when user input is detected, and reverting to the user's intended scale after a cooldown period.

**Inputs and Outputs:**
- **Inputs:** Current user interaction state (active/inactive), timestamp of last interaction, target render scale (`userRenderScale`), interaction render scale (`interactionRenderScale`), and interaction hold duration (`INTERACTION_HOLD_MS`).
- **Outputs:** Effective render scale multiplier applied to the rendering engine.

**Data Structures:**
- `interactionRenderScale`: A floating-point value in the range $[0, 1]$. A value of $0$ disables scaling (full resolution), while $1$ represents the lowest resolution [S1, unofficial].
- `userRenderScale`: The target render scale intended by the user when not interacting [S1, unofficial].
- `lastMoveTime`: A timestamp (e.g., milliseconds since epoch) recording the most recent user interaction event [S1, unofficial].
- `INTERACTION_HOLD_MS`: A constant defining the delay before resolution is restored (e.g., $200$ ms) [S1, unofficial].

**Algorithm Steps:**
1. **Detect Interaction:** Check if the user is currently interacting (e.g., moving the camera).
2. **Update Timestamp:** If interaction is detected, update `lastMoveTime` to the current time.
3. **Apply Damping:** If interaction is active, set the effective render scale to `interactionRenderScale`.
4. **Check Cooldown:** If interaction is inactive, calculate the elapsed time since `lastMoveTime`.
   - If `currentTime - lastMoveTime < INTERACTION_HOLD_MS`, continue using `interactionRenderScale`.
   - Otherwise, revert the effective render scale to `userRenderScale`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Configuration
INTERACTION_HOLD_MS = 200
interactionRenderScale = 0.25  # Reduce resolution to 25% during interaction
userRenderScale = 1.0          # Target full resolution
lastMoveTime = 0
isInteracting = False

def get_effective_render_scale(current_time, interaction_active):
    global lastMoveTime, isInteracting
    
    # Update interaction state and timestamp
    if interaction_active:
        isInteracting = True
        lastMoveTime = current_time
    else:
        isInteracting = False
        
    # Determine effective scale
    if isInteracting:
        return interactionRenderScale
    else:
        # Check if hold time has passed
        if (current_time - lastMoveTime) >= INTERACTION_HOLD_MS:
            return userRenderScale
        else:
            return interactionRenderScale
```

**Parameters:**
- `interactionRenderScale`: Typical value $0.25$ [S1, unofficial].
- `INTERACTION_HOLD_MS`: Typical value $200$ ms [S1, unofficial].

**Edge Cases:**
- If `interactionRenderScale` is set to $0$, scaling is disabled, and the renderer may drop frames during interaction [S1, unofficial].
- Rapid, intermittent interactions may cause the resolution to oscillate if the hold time is too short.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ for the scale determination logic. The rendering cost scales with the square of the resolution reduction factor (for 2D projection) or cube (for 3D volume processing), but the damping only affects the final image resolution, not the underlying volume traversal complexity in all implementations [S1, unofficial].
- **Memory Complexity:** $O(1)$ additional memory for state tracking.
- **Strengths:** Maintains interactive frame rates during navigation; seamless transition back to high quality.
- **Limitations:** Requires careful tuning of `INTERACTION_HOLD_MS` to avoid visual popping; does not reduce computational load of volume traversal itself, only the final compositing/resolution [S1, unofficial].

## Background
The technique is documented in the context of web-based, out-of-core volume rendering engines, specifically within the "Kiln" renderer project, which targets multi-GB datasets using WebGPU [S1, unofficial]. It addresses the latency issues inherent in high-resolution volume rendering during interactive exploration.

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
