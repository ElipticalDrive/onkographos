# Adaptive Sampling for Interaction

Also called: Adaptive Quality for Volume Rendering, Adaptive Sampling for Interactive Volume Rendering, Dynamic Sampling Rate, Interaction-aware Adaptive Sampling, Two-rate Interaction Sampling

## Overview
Adaptive Sampling for Interaction is a technique to maintain smooth user interaction with large volume datasets by dynamically adjusting the rendering step size based on framerate, and performing a high-quality pass after interaction stops [S1, unofficial]. It lowers the sampling rate during user interaction to maintain performance, then re-renders at a higher rate once the action is completed [S2][S3]. Unlike static resolution scaling, it is time-bound and context-aware, specifically designed to decouple performance from user input latency [S1, unofficial]. It builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md).

## How it works
**Inputs and Outputs:**
*   **Input:** User interaction events (e.g., camera movement), current frame rate metrics, volume dataset, transfer function.
*   **Output:** Rendered image with adaptive sampling density; updated step size parameters.

**Data Structures:**
*   `RenderQualityChange`: An enum or event type indicating the quality state. Values include `MAX` (high quality) and potentially lower states (coarse).
*   `min_step_size`: A scalar float representing the minimum allowed step size for the volume renderer. Clamped between 0.1 and 1.0 to satisfy library constraints (e.g., Vispy's `VolumeVisual` raises a `ValueError` if `relative_step_size` < 0.1) [S1, unofficial].
*   `relative_step_size`: The current step size multiplier used by the renderer.

**Algorithm:**
The method operates as a control loop triggered by user interaction and frame timing.

1.  **Event Emission:** When a quality change is triggered (e.g., `change_render_quality` is called), the base layer emits a `render_quality` event [S1, unofficial].
2.  **Event Handling:** The visualization layer (e.g., `VispyImageLayer`) listens for the `render_quality` event [S1, unofficial].
3.  **Quality Adjustment:**
    *   If the event indicates `RenderQualityChange.MAX` (interaction stopped or high quality requested), set `relative_step_size` to `min_step_size` [S1, unofficial].
    *   If the framerate drops below a desired threshold during interaction, increase `relative_step_size` (coarser sampling) [S1, unofficial].
    *   If the framerate is very high, decrease `relative_step_size` (finer sampling) [S1, unofficial].
4.  **Step Size Update:** Update `min_step_size` based on current performance to ensure it stays within valid bounds and allows for future refinement. A typical update rule observed is:
    $$ \text{min\_step\_size} = \text{clip}\left(\frac{\text{new\_step\_size}}{4}, 0.1, 1.0\right) $$
    where `new_step_size` is the current step size before adjustment [S1, unofficial]. The lower bound is strictly 0.1 due to library constraints [S1, unofficial].
5.  **Final Pass:** When the user stops moving the camera for a sufficient duration, trigger a final high-quality render pass using the minimum step size [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class AdaptiveInteractionSampler:
    def __init__(self, min_step_size=0.1, max_step_size=1.0):
        self.min_step_size = min_step_size
        self.max_step_size = max_step_size
        self.current_step_size = 0.5  # Initial guess
        self.interaction_timeout = 1.0  # Seconds to wait before high-quality pass
        self.last_interaction_time = 0.0

    def on_user_interaction(self, timestamp):
        """Called when user moves camera or adjusts parameters."""
        self.last_interaction_time = timestamp
        # During interaction, allow coarser sampling if needed
        # Logic to increase step size based on framerate drop would go here
        pass

    def on_framerate_check(self, current_fps, target_fps):
        """Called every frame to adjust sampling based on performance."""
        if current_fps < target_fps:
            # Increase step size (coarser) to improve performance
            self.current_step_size = min(self.current_step_size * 1.1, self.max_step_size)
        else:
            # Decrease step size (finer) if performance is good
            self.current_step_size = max(self.current_step_size * 0.9, self.min_step_size)
        
        # Clamp to valid range
        self.current_step_size = max(0.1, min(self.current_step_size, 1.0))
        return self.current_step_size

    def on_interaction_stopped(self, timestamp):
        """Called when user stops interacting for a duration."""
        if timestamp - self.last_interaction_time > self.interaction_timeout:
            # Trigger high-quality pass
            self.current_step_size = self.min_step_size
            # Update min_step_size for future interactions based on current performance
            # Example: set min to 1/4 of current step size, clamped
            self.min_step_size = max(0.1, min(self.current_step_size / 4, 1.0))
            return True  # High quality pass triggered
        return False

    def get_step_size(self):
        return self.current_step_size
```

**Parameters:**
*   `min_step_size`: The minimum allowed step size. Clamped between 0.1 and 1.0 [S1, unofficial].
*   `High sampling rate`: 6 to 10 samples per voxel [S2][S3].
*   `Target FPS`: Not explicitly defined in sources, but implied as a threshold for triggering coarser sampling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Library Constraints:** Vispy's `VolumeVisual` raises a `ValueError` if `relative_step_size` is set below 0.1. The implementation must clamp `min_step_size` to at least 0.1 [S1, unofficial].
*   **Attenuated MIP Artifacts:** With `attenuated_mip`, changing step size causes brightness fluctuations (de-attenuation) during rotation. The volume may appear suddenly brighter and then dim again after the high-quality pass stops movement [S1, unofficial].
*   **Performance Trade-off:** Without adaptive quality, `attenuated_mip` renders at 17-30 fps. With adaptive quality, it reaches 60 fps during interaction, but the final high-quality pass takes 2-3 seconds [S1, unofficial].

## Subtopics

### [Interaction Render Scale Damping](8891-interaction-render-scale-damping.md)
A strategy that dynamically reduces render resolution during user interaction to maintain a target frame rate, then restores original resolution after interaction ceases. It differs by being time-bound and context-aware, decoupling performance from user input latency. It builds on this method.

## Operating characteristics
*   **Time Complexity:** Frame updates during interaction are significantly faster than high-quality rendering. High sampling rate (6-10 samples/voxel) takes nearly two seconds on Octane2 and nearly one second on GeForce3 [S2][S3]. With adaptive quality (decreased step size), frame rates reach 60 fps during interaction [S1, unofficial].
*   **Memory Complexity:** Not explicitly defined, but depends on the volume dataset size and the number of active samples per ray.
*   **Scaling:** Performance scales with the number of samples per voxel. Lowering the sample rate during interaction reduces the number of volume queries, improving frame rate.
*   **Strengths:** Maintains interactivity during camera movement or parameter adjustment. Provides effective feedback for guiding user exploration, as surface size and location remain readily apparent even with coarser surfaces [S2][S3].
*   **Limitations:** The final high-quality pass can take 2-3 seconds, causing a delay in visual feedback after interaction stops [S1, unofficial]. Attenuated MIP rendering suffers from brightness fluctuations during step size changes [S1, unofficial].
*   **When to Use:** Interactive volume rendering where user exploration (camera movement, transfer function adjustment) is frequent and requires immediate feedback.
*   **When Not to Use:** When final visual fidelity is required without delay, or when the rendering pipeline does not support dynamic step size adjustment (e.g., fixed-step hardware limitations).

## Background
The concept of lowering sampling rates during interaction to maintain performance, and re-rendering at higher rates after actions are completed, was introduced in the context of interactive volume rendering using multi-dimensional transfer functions [S2][S3]. This approach allows users to explore complex volume datasets with 3D transfer functions without being hindered by long frame update times associated with high-quality rendering (e.g., 6-10 samples per voxel) [S2][S3]. The technique was further implemented in modern visualization frameworks like Napari, using Vispy for rendering, to provide a seamless user experience with large volumes [S1, unofficial].

## Sources
[S1] Adaptive quality for volume rendering, https://github.com/napari/napari/pull/6930, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S3] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
