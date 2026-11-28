# VTK Frame Rate Targeting & Adaptive Sampling

Also called: AutoAdjustSampleDistances, SetDesiredUpdateRate

## Overview
This method maintains a target frame rate in volume rendering by dynamically adjusting the sampling distance per frame based on an allocated render time budget [S1]. It differs from local error-driven adaptive methods by using a global time-budgeting controller that distributes frame time among scene objects rather than computing per-pixel error metrics [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Desired frame rate $F_{\text{target}}$, scene object list, volume dataset.
*   **Output:** Adjusted sampling distance per volume object, rendered image.

**Data Structures:**
*   Render window controller: Tracks frame timing and distributes time budgets to scene objects [S1].
*   Volume ray-cast mapper: Stores the `AutoAdjustSampleDistances` flag and the current sampling distance parameter [S1].

**Parameters:**
*   $F_{\text{target}}$: The desired frame rate for the render engine. In VR applications requiring 90 fps per eye across three views (left eye, right eye, screen preview), a VTK target of 300 fps is used to account for overhead and multiple views [S1].
*   `AutoAdjustSampleDistances`: Boolean flag on the volume ray-cast mapper that enables automatic sampling distance adjustment [S1].

**Steps:**
1.  **Set Target Rate:** Configure the render window controller with the desired frame rate $F_{\text{target}}$ [S1].
2.  **Calculate Budget:** The controller computes the per-frame time budget $T_{\text{frame}} = 1 / F_{\text{target}}$ [S1].
3.  **Distribute Time:** The total frame time is divided among all scene objects, so each object receives a smaller allocated time as more objects are added [S1].
4.  **Adapt Sampling:** For each volume mapper with automatic adjustment enabled:
    *   The mapper tracks its allocated render time [S1].
    *   It automatically adjusts its internal sampling distance to ensure rendering completes within the allocated time slice [S1].
    *   If rendering exceeds the budget, the sampling distance increases to reduce the sample count; if it finishes early, the distance may decrease to improve quality, though the primary mechanism prevents overshoot [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import time

def render_loop(target_fps, scene_objects):
    frame_budget = 1.0 / target_fps
    while True:
        time_per_object = frame_budget / len(scene_objects)
        for obj in scene_objects:
            start = time.perf_counter()
            # Volume integration step (delegated to parent method)
            obj.render_volume()
            elapsed = time.perf_counter() - start
            if obj.auto_adjust_samples:
                # Exact adaptation law not specified in sources [G]
                if elapsed > time_per_object:
                    obj.sampling_distance *= 1.1
                elif elapsed < time_per_object * 0.8:
                    obj.sampling_distance *= 0.9
        yield rendered_image
```

**Edge Cases and Pitfalls:**
*   **Banding Artifacts:** Enforcing a strict frame rate target causes visible banding artifacts due to reduced sampling quality [S1].
*   **Proximity Issues:** Without a target, frame rate drops significantly (e.g., to 40 fps) when the volume is very close to the viewer because more pixels are filled, requiring more rays and higher computational cost [S1].

## Operating characteristics
*   **Time Complexity:** The rendering time is bounded by the allocated frame budget, but the complexity of the volume integral varies with the adaptive sampling distance [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Without a frame rate target, performance drops to 40 fps when the volume is very close to the viewer [S1]. With a 300 fps target, frame rate often exceeds 100 fps, maintaining viewer comfort [S1].
*   **Strengths:** Guarantees a minimum frame rate, preventing uncomfortable drops in interactive VR scenarios [S1].
*   **Limitations:** Visual quality degrades (banding) as sampling distance increases to meet time constraints [S1].

## Subtopics

### PI Controller for Frame Rate Regulation
A Proportional-Integral controller that dynamically adjusts the image-space sub-sampling distance $D_I$ to maintain a target frame rate during interactive sessions [S2]. It successfully maintains a steady 25 fps target during remote visualization, despite navigation-induced workload spikes [S2].

## Background
This technique is part of the VTK library's support for interactive volume rendering, specifically designed for VR applications where consistent frame rates are critical for user comfort [S1]. It was documented in the context of integrating VTK with Unity for 3D medical imaging visualization [S1].

## Sources
[S1] Virtual interaction and visualisation of 3D medical imaging data with VTK and Unity, https://doi.org/10.1049/htl.2018.5064, fetched 2026-10-28
[S2] Efficient Raycasting of Volumetric Depth Images for Remote Visualization of Large Volumes at High Frame Rates, https://doi.org/10.1109/pacificvis56936.2023.00014, fetched 2026-10-24
