# Time-critical level-of-detail selection

Also called: automatic level-of-detail selection

## Overview
Time-critical level-of-detail selection is an adaptive rendering technique that dynamically adjusts the visual fidelity of a volume rendering to maintain a target frame rate [S1, unofficial]. It solves the problem of unpredictable rendering times in complex scenes by predicting the cost of the next frame and selecting an appropriate level-of-detail (LOD) beforehand [S1, unofficial]. Unlike static LOD schemes, this method is driven by real-time performance statistics, allowing it to react to scene complexity changes or hardware load variations [S1, unofficial]. It differs from [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md) by adjusting the geometric or shader complexity of the volume representation rather than the pixel resolution of the output viewport.

## How it works
The method operates in a feedback loop per frame.

**Inputs:**
*   Current frame render time ($T_{current}$).
*   Desired frame rate ($R_{target}$), which defines the time budget $T_{budget} = 1/R_{target}$ [S1, unofficial].
*   Performance statistics history (used for prediction) [S1, unofficial].

**Outputs:**
*   Selected Level-of-Detail ($LOD_{next}$) for the subsequent frame [S1, unofficial].

**Data Structures:**
*   **Performance Buffer:** A circular buffer or moving average window storing recent frame times ($T_i$). The size is not specified in the sources [G].
*   **LOD Map:** A mapping from predicted time budgets to discrete LOD levels (e.g., voxel spacing, transfer function complexity, or proxy geometry type).

**Steps:**
1.  **Collect Statistics:** Measure the actual rendering time $T_{current}$ of the current frame [S1, unofficial].
2.  **Predict Time:** Estimate the rendering time for the next frame, $T_{pred}$, using the collected performance statistics. The sources do not specify the exact prediction algorithm (e.g., exponential moving average, linear regression) [G]. A common implementation is an exponential moving average: $T_{pred} = \alpha T_{current} + (1-\alpha) T_{pred}$, where $\alpha$ is a smoothing factor [G].
3.  **Select LOD:** Compare $T_{pred}$ against $T_{budget}$ [S1, unofficial].
    *   If $T_{pred} > T_{budget}$, select a lower LOD (simpler representation) to reduce future render time.
    *   If $T_{pred} \le T_{budget}$, select a higher LOD (more detailed representation) to improve visual quality, provided it does not exceed the budget significantly.
4.  **Apply LOD:** Configure the renderer for the next frame with the selected $LOD_{next}$ [S1, unofficial].

**Parameters:**
*   **Desired Frame Rate ($R_{target}$):** The target frames per second. Typical values depend on the application (e.g., 30 FPS for interactive, 60 FPS for VR) [S1, unofficial].
*   **Prediction Smoothing Factor ($\alpha$):** Controls the responsiveness to sudden changes. Not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Oscillation:** Rapid switching between LOD levels if $T_{pred}$ hovers near $T_{budget}$. Mitigation requires hysteresis or a minimum dwell time at each LOD level [G].
*   **Artifacts:** Switching LODs can cause visual discontinuities. The sources note that using different proxy geometries other than planes can avoid artifacts but incurs a computational price [S1, unofficial]. Alternatively, modifying transfer functions in lower-resolution nodes can limit artifacts [S1, unofficial].
*   **Prediction Error:** If the prediction model is inaccurate (e.g., due to sudden scene changes), the selected LOD may be inappropriate, leading to frame rate drops or wasted quality [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def time_critical_lod_selection(frame_time, target_fps, history, lod_map):
    """
    frame_time: float, time taken to render the current frame in seconds.
    target_fps: float, desired frames per second.
    history: list of floats, recent frame times.
    lod_map: dict or function, maps predicted time budget to LOD index.
    
    Returns:
    int, the selected LOD index for the next frame.
    """
    # Update history with current frame time
    history.append(frame_time)
    
    # Predict next frame time using exponential moving average [G]
    alpha = 0.5  # Smoothing factor, typical value [G]
    if len(history) == 1:
        predicted_time = frame_time
    else:
        # Use the last predicted time or average of history? 
        # Standard EMA uses previous prediction. 
        # Here we assume history stores raw times, so we compute EMA on the fly.
        # For simplicity, let's use average of last N frames if N is small, 
        # or EMA. Let's stick to EMA as it's common for this [G].
        # Note: The source doesn't specify the predictor. 
        # We implement a simple EMA based on the last prediction.
        # However, since we don't have the previous prediction stored in the args,
        # we approximate using the average of the last few frames or assume 
        # the 'history' is maintained externally. 
        # To be self-contained, we compute a simple average of the last K frames [G].
        k = 5  # Window size, typical value [G]
        recent_times = history[-k:]
        predicted_time = sum(recent_times) / len(recent_times)

    # Calculate time budget
    time_budget = 1.0 / target_fps

    # Select LOD based on predicted time vs budget
    # LOD map should handle the mapping. 
    # If predicted_time > budget, we need lower LOD (higher index if 0 is highest quality? 
    # Or lower index? Let's assume LOD index 0 is highest detail, N is lowest.
    # So if time is high, we want higher index.
    
    selected_lod = lod_map.predict_lod(predicted_time, time_budget)
    
    return selected_lod
```

## Subtopics

### Automatic Shader Level-of-Detail
A method by Olano et al. that performs automatic level-of-detail selection specifically for shaders using an automatic system for shader simplification [S2]. It sets itself apart by operating at the shader code level rather than the geometric or voxel resolution level, allowing for dynamic simplification of procedural noise or texture functions based on rendering time constraints [S2]. It is chosen when shader complexity is the bottleneck rather than geometry or volume traversal [S2].

## Operating characteristics
*   **Time Complexity:** The selection logic itself is $O(1)$ or $O(N)$ where $N$ is the history window size, negligible compared to rendering [S1, unofficial]. The rendering time varies with the selected LOD.
*   **Memory Complexity:** Requires storage for performance history and LOD maps. Not given in the sources [G].
*   **Measured Performance:** Not given in the sources [G].
*   **Strengths:** Maintains target frame rate in dynamic scenes; adapts to hardware variations; allows user-defined quality preferences via parameters [S1, unofficial].
*   **Limitations:** Potential for visual artifacts during LOD transitions; prediction errors can lead to suboptimal LOD selection; requires careful tuning of prediction parameters [S1, unofficial].
*   **When to Use:** Interactive volume visualization where frame rate stability is critical [S1, unofficial].
*   **When Not to Use:** Offline rendering where quality is prioritized over speed; scenes with highly static complexity where static LOD is sufficient [S1, unofficial].

## Background
The concept was developed by Li et al. as part of adaptive volume rendering schemes [S1, unofficial]. It builds on earlier work by Aliaga et al. regarding depth-based adaptive rendering [S1, unofficial]. The method is cited in the context of large-scale displays and handheld touchscreen interaction for volume visualization [S1, unofficial].

## Sources
[S1] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S2] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
