# Adaptive Sampling for Interaction

Also called: Adaptive Sampling for Interactive Volume Rendering, Dynamic Sampling Rate, Interaction-aware Adaptive Sampling, Two-rate Interaction Sampling

## Overview
Adaptive Sampling for Interaction is a performance optimization technique that dynamically lowers the sampling rate during user interaction (such as camera movement or transfer function editing) to maintain interactive frame rates, then restores the high sampling rate once the interaction is complete [S1][S2]. Unlike [Adaptive Sampling](1385-adaptive-sampling.md), which varies sampling based on local scene complexity or error estimates, this method varies sampling based on global user input state [S1][S2]. It builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by applying a coarse, uniform step size during interaction and a fine, potentially adaptive step size during idle rendering [S1][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** User interaction state (active/inactive), current sampling parameters, volume data.
*   **Output:** Rendered image with appropriate sampling density.

**Data Structures:**
*   **Interaction State:** A boolean flag or timestamp indicating if the user is currently interacting.
*   **Sampling Rate Parameters:**
    *   `high_sample_rate`: Samples per voxel during idle rendering. Typical value: 6–10 [S1][S2].
    *   `low_sample_rate`: Samples per voxel during interaction. Not given in the sources.
    *   `hold_time`: Delay in milliseconds before restoring high sampling rate after interaction stops. Typical value: 200 ms [S3, unofficial].

**Steps:**
1.  **Detect Interaction:** Monitor user input events (mouse, keyboard, UI controls). Set `interaction_active = true` on event start.
2.  **Select Sampling Rate:**
    *   If `interaction_active` is true, set `current_sample_rate = low_sample_rate`.
    *   If `interaction_active` is false, set `current_sample_rate = high_sample_rate`.
3.  **Ray Marching:** Perform volume rendering using the selected `current_sample_rate`. The step size $\Delta t$ is derived from this rate. For a voxel of size $v$, $\Delta t = v / \text{current\_sample\_rate}$.
4.  **Interaction Timeout:** If `interaction_active` is true, check if the time since the last interaction event exceeds `hold_time`. If so, set `interaction_active = false` to trigger the switch to `high_sample_rate` on the next frame.
5.  **Compositing:** Use front-to-back compositing as defined in the parent method [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state
interaction_active = False
last_interaction_time = 0
hold_time_ms = 200  # [S3, unofficial]
high_sample_rate = 8  # Typical value from [S1][S2]
low_sample_rate = 2   # Not given in the sources, assumed lower than high_sample_rate

def on_user_input():
    global interaction_active, last_interaction_time
    interaction_active = True
    last_interaction_time = current_time_ms()

def get_sample_rate():
    if interaction_active:
        return low_sample_rate
    else:
        return high_sample_rate

def render_frame():
    # Check for interaction timeout
    if interaction_active:
        if current_time_ms() - last_interaction_time > hold_time_ms:
            interaction_active = False
    
    sample_rate = get_sample_rate()
    step_size = voxel_size / sample_rate
    
    # Perform ray marching with step_size
    # ... (see parent method for compositing details)
```

**Edge Cases and Pitfalls:**
*   **Visual Quality:** During interaction, the rendered surface appears coarser. However, surface size and location remain readily apparent, providing effective feedback for guiding user exploration [S1][S2].
*   **Precision:** Using integer sample rates (e.g., 6 vs 10) is sufficient for this method [S1][S2].

## Subtopics

### Interaction Render Scale Damping
Dynamically reduces render resolution (not just sampling rate) during camera interaction to maintain frame rate, then restores it after interaction stops [S3, unofficial]. It uses an `interactionRenderScale` multiplier (e.g., 0.25) to reduce resolution during movement and a `userRenderScale` for the target resolution [S3, unofficial]. A `INTERACTION_HOLD_MS` (200 ms) delay prevents flickering when the user pauses [S3, unofficial]. This is distinct from sampling rate reduction as it affects the output image dimensions directly [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces rendering time during interaction by lowering the number of samples per ray. The exact speedup depends on the ratio of `high_sample_rate` to `low_sample_rate`.
*   **Memory Complexity:** No additional memory overhead beyond storing the interaction state and timing variables.
*   **Measured Performance:** At high sampling rates (6–10 samples per voxel), frame updates can take nearly two seconds on an Octane2 and nearly one second on a GeForce3 [S1][S2]. Lowering the sample rate during interaction significantly reduces this time, enabling interactivity.
*   **Strengths:** Simple to implement; effective at maintaining interactivity during expensive operations like transfer function editing.
*   **Limitations:** Visual quality degrades during interaction; requires a "hold" period to avoid premature restoration of high quality.
*   **When to Use:** When user interaction (camera, transfer function) is frequent and expensive to render at full quality.
*   **When Not to Use:** When visual fidelity during interaction is critical.

## Background
This technique was introduced in the context of interactive volume rendering using multi-dimensional transfer functions [S1][S2]. The authors noted that high sampling rates (6–10 samples per voxel) were necessary for smooth boundary rendering but resulted in frame times of nearly one to two seconds on contemporary hardware (Octane2, GeForce3) [S1][S2]. To address this, they proposed lowering the sample rate during interaction and restoring it afterward [S1][S2]. The concept of dynamically adjusting resolution/sampling based on user activity has also been implemented in modern web-based volume renderers like Kiln [S3, unofficial].

## Sources
[S1] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] Interactive Volume Rendering Using Multi-Dimensional Transfer Functions, https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
[S3] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
