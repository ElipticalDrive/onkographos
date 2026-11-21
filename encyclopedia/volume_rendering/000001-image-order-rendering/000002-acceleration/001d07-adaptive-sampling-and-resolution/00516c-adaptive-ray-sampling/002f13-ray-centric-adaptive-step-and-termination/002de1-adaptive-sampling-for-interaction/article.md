# Adaptive Sampling for Interaction

Also called: Adaptive Quality for Volume Rendering, Adaptive Sampling for Interactive Volume Rendering, Dynamic Sampling Rate, Interaction-aware Adaptive Sampling, Two-rate Interaction Sampling

## Overview
Adaptive Sampling for Interaction is a technique to maintain smooth user interaction with large volume datasets by dynamically lowering the rendering sampling rate during camera movement or parameter adjustment, then restoring high-quality rendering after the interaction ceases [S1, unofficial][S2, S3][S4, unofficial]. It is triggered by user activity rather than a fixed timer [S1, unofficial][S2]. Unlike purely error-based adaptive sampling, it prioritizes temporal responsiveness over local data complexity [S1, unofficial][S2]. This method bridges the gap between interactive responsiveness and final image quality, ensuring that coarse feedback is provided during exploration while preserving detail for static inspection [S2, S3].

## How it works
The method operates as a two-state control loop that modulates the sampling density based on the state of user interaction.

### Inputs and Outputs
*   **Inputs:** User interaction events (e.g., camera rotation, translation, zoom), current frame rate metrics, and target performance thresholds [S1, unofficial][S2].
*   **Outputs:** Adjusted `relative_step_size` for the volume renderer and a final high-quality render pass upon interaction completion [S1, unofficial][S2].

### Data Structures
The implementation typically involves an event listener structure:
1.  **Base Layer:** Emits a `render_quality` event when the quality state changes [S1, unofficial].
2.  **VispyImageLayer (or equivalent):** Listens to the `render_quality` event and updates the rendering node's `relative_step_size` [S1, unofficial].
3.  **Volume Visual Node:** Holds the `relative_step_size` attribute, which is passed to the renderer for ray marching [S1, unofficial].

### Parameters
*   **High Sampling Rate:** Typically 6 to 10 samples per voxel [S2, S3][S4, unofficial].
*   **Min Step Size:** The minimum allowed value for `relative_step_size`. In Vispy implementations, this is clamped to a minimum of 0.1 because the underlying `VolumeVisual` raises a `ValueError` if the value is lower [S1, unofficial].
*   **Interaction Step Scale:** A constant defining the coarse step size multiplier during interaction (e.g., `INTERACTION_STEP_SCALE` in webpic) [S5, unofficial].
*   **Interaction Render Scale:** A constant defining the coarse resolution multiplier during interaction (e.g., `INTERACTION_RENDER_SCALE` in webpic) [S5, unofficial].

### Algorithm Steps
1.  **Initialization:** Set the initial `relative_step_size` to the default high-quality value corresponding to 6–10 samples per voxel [S2, S3].
2.  **Interaction Detection:** Monitor for user input events (camera movement, parameter changes) [S1, unofficial][S5, unofficial].
3.  **Coarsening (During Interaction):**
    *   When interaction begins, immediately increase the `relative_step_size` (decrease sampling density) to a coarse level defined by `INTERACTION_STEP_SCALE` [S5, unofficial].
    *   If using resolution scaling, simultaneously reduce the render resolution using `INTERACTION_RENDER_SCALE` [S5, unofficial].
    *   Ensure `relative_step_size` is clamped to $\ge 0.1$ to prevent runtime errors [S1, unofficial].
4.  **Monitoring:** Optionally monitor frame rate. If the frame rate drops below a threshold, further increase step size; if it is very high, decrease it [S1, unofficial].
5.  **Interaction End Detection:** Detect when the user has stopped interacting for a specified idle duration [S1, unofficial].
6.  **Restoration (Post-Interaction):**
    *   **Instant Restore:** Set `relative_step_size` back to the minimum allowed value (highest quality) [S1, unofficial].
    *   **Ramped Restore (Variant):** Smoothly restore quality over multiple frames to prevent visual popping. Set step scale to an intermediate value (e.g., 0.7) and incrementally approach 1.0 over subsequent frames [S5, unofficial].
    *   Execute a final high-quality render pass [S1, unofficial][S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class AdaptiveInteractionSampler:
    def __init__(self, renderer, min_step_size=0.1, high_quality_step=0.1, 
                 interaction_step_scale=2.0, idle_threshold=1.0):
        self.renderer = renderer
        self.min_step_size = min_step_size
        self.high_quality_step = high_quality_step
        self.interaction_step_scale = interaction_step_scale
        self.idle_threshold = idle_threshold
        self.is_interacting = False
        self.idle_time = 0.0

    def on_interaction_start(self):
        """Called when user begins camera movement or parameter adjustment."""
        self.is_interacting = True
        self.idle_time = 0.0
        # Immediately coarsen sampling
        coarse_step = self.high_quality_step * self.interaction_step_scale
        # Clamp to Vispy constraint
        coarse_step = max(coarse_step, 0.1)
        self.renderer.set_relative_step_size(coarse_step)

    def on_interaction_update(self, dt):
        """Called every frame during interaction."""
        if not self.is_interacting:
            return
        self.idle_time = 0.0
        # Optional: Adjust step size based on current framerate
        # current_step = self.renderer.get_relative_step_size()
        # if framerate < target: current_step *= 1.1
        # self.renderer.set_relative_step_size(current_step)

    def on_interaction_end(self):
        """Called when user stops interacting."""
        self.is_interacting = False
        # Restore high quality
        self.renderer.set_relative_step_size(self.high_quality_step)
        # Trigger final high-quality render pass
        self.renderer.render()

    def update(self, dt):
        """Main loop update."""
        if not self.is_interacting:
            self.idle_time += dt
            # Check if idle long enough to trigger restoration
            if self.idle_time >= self.idle_threshold:
                self.on_interaction_end()
```

### Edge Cases and Pitfalls
*   **Vispy Constraint:** The `VispyVolumeVisual` raises a `ValueError` if `relative_step_size` is set below 0.1. The implementation must clamp the minimum step size to at least 0.1 [S1, unofficial].
*   **Attenuated MIP Brightness Fluctuation:** When using attenuated maximum intensity projection (attenuated_mip), changing the step size can cause brightness fluctuations (de-attenuation) during rotation. The volume may appear suddenly brighter during interaction and dim again after the high-quality pass stops movement [S1, unofficial].
*   **Visual Popping:** Instantly restoring high quality can cause a visible "pop" in resolution. A ramped restoration strategy mitigates this [S5, unofficial].

## Subtopics

### [Interaction Render Scale Damping](8891-interaction-render-scale-damping.md)
Interaction Render Scale Damping dynamically reduces the render resolution during camera interaction to maintain frame rate, then restores it after interaction stops. It temporarily overrides the user's intended render scale with a lower multiplier during interaction, reverting to the target scale after a fixed delay.

### Interaction Quality Ramp
Interaction Quality Ramp is a technique to smoothly restore rendering quality after user interaction by ramping step and render scales over multiple painted frames instead of instantly, preventing visual popping [S5, unofficial]. On gesture start, it immediately sets step scale to `INTERACTION_STEP_SCALE` and render scale to `INTERACTION_RENDER_SCALE`. On gesture end, it initiates a settle ramp: the first level sets step scale to 0.7 and render scale to 1. Each painted frame advances the ramp exactly one level until full quality (scale 1) is reached. If a new gesture starts mid-ramp, it immediately re-enters interaction quality, interrupting the restore sequence. Fly mode uses full resolution render scale but coarser step scale (0.7); on idle, it settles to full quality without changing render scale. Marker drag triggers interaction quality on the active edge; release falls back to fly tier if fly is live, otherwise follows the settle ramp [S5, unofficial].

## Operating characteristics
*   **Performance:** At high sampling rates (6–10 samples per voxel), frame updates can take nearly two seconds on an Octane2 and nearly one second on a GeForce3 [S2, S3][S4, unofficial]. By lowering the sample rate during interaction, frame rates can be maintained at 60 fps, compared to 17–30 fps without adaptive quality for attenuated_mip [S1, unofficial].
*   **Visual Fidelity:** During interaction, the volume rendered surface appears coarser. However, the surface size and location remain readily apparent, providing effective feedback for guiding user exploration [S2, S3][S4, unofficial].
*   **Trade-offs:** The final high-quality pass after interaction stops can take 2–3 seconds, creating a latency between the user's intent and the final visual result [S1, unofficial]. This is acceptable for exploration but may be perceived as sluggish if the user expects immediate high-fidelity feedback.
*   **Hardware Dependency:** Performance gains are significant on older or less powerful hardware (e.g., Octane2, GeForce3) where high-resolution volume rendering is expensive [S2, S3].

## Background
The technique was introduced in the context of interactive volume rendering using multi-dimensional transfer functions, where high sampling rates were necessary to smooth boundaries between materials but caused unacceptable latency during user interaction [S2, S3][S4, unofficial]. The approach was later implemented in modern visualization frameworks like Napari and Webpic to handle large datasets interactively [S1, S5, unofficial].

## Sources
[S1] Adaptive quality for volume rendering, https://github.com/napari/napari/pull/6930, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S3] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
[S4] https://www.researchgate.net/publication/2399866_Interactive_Volume_Rendering_Using_Multi-Dimensional_Transfer_Functions_and_Direct_Manipulation_Widgets, https://www.researchgate.net/publication/2399866_Interactive_Volume_Rendering_Using_Multi-Dimensional_Transfer_Functions_and_Direct_Manipulation_Widgets, fetched 2026-11-01 (unofficial)
[S5] rusaitis/webpic, https://github.com/rusaitis/webpic, fetched 2026-10-27 (unofficial)
