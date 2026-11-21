# VTK Frame Rate Targeting & Adaptive Sampling

Also called: AutoAdjustSampleDistances, SetDesiredUpdateRate

## Overview
This method maintains a target frame rate in VTK volume rendering by dynamically adjusting the sampling distance per frame based on the allocated render time budget [S1]. It sets itself apart from static adaptive methods by using a global time-budgeting controller that distributes frame time among scene props, rather than local error metrics [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Desired frame rate ($F_{target}$), scene geometry (props), volume data.
*   **Output:** Adjusted sampling distance per volume prop, rendered image.

**Data Structures:**
*   `vtkExternalOpenGLRenderWindow`: Manages the render loop and frame timing.
*   `vtkOpenGLGPUVolumeRayCastMapper`: Contains the `AutoAdjustSampleDistances` flag and the current sampling distance parameter.

**Parameters:**
*   $F_{target}$: The desired frame rate for the VTK render engine. In VR applications with 3 views (left eye, right eye, screen preview), a target of 90 fps per eye requires a VTK target of approximately 300 fps to account for overhead and multiple views [S1].
*   `AutoAdjustSampleDistances`: Boolean flag on `vtkOpenGLGPUVolumeRayCastMapper` [S1].

**Steps:**
1.  **Set Target Rate:** Call `SetDesiredUpdateRate(F_{target})` on the `vtkExternalOpenGLRenderWindow` instance [S1].
2.  **Calculate Budget:** VTK calculates the per-frame time budget $T_{frame} = 1 / F_{target}$ [S1].
3.  **Distribute Time:** The total frame time is divided among all props in the scene. As more props are added, the allocated time for each prop decreases [S1].
4.  **Adapt Sampling:** For each `vtkOpenGLGPUVolumeRayCastMapper` with `AutoAdjustSampleDistances` enabled:
    *   The mapper monitors its allocated render time.
    *   It automatically adjusts its internal sampling distance (step size) to ensure the volume renders within its allocated time slice [S1].
    *   If the render takes too long, the sampling distance increases (fewer samples). If it finishes early, the sampling distance may decrease (more samples), though the primary driver is preventing overshoot [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
render_window = vtkExternalOpenGLRenderWindow()
# Set target frame rate. For VR with 3 views (L, R, Preview) targeting 90fps per view:
# VTK target is set higher to account for overhead.
render_window.SetDesiredUpdateRate(300) 

# Volume Setup
volume_mapper = vtkOpenGLGPUVolumeRayCastMapper()
volume_mapper.SetInputData(volume_data)
# Enable automatic adjustment of sampling distance based on time budget
volume_mapper.AutoAdjustSampleDistancesOn()

# Render Loop
while not done:
    # VTK internal loop handles:
    # 1. Calculating time budget per frame (1/300 s)
    # 2. Distributing budget among props
    # 3. Adjusting volume_mapper's sampling distance if needed
    render_window.Render()
```

**Edge Cases and Pitfalls:**
*   **Banding Artifacts:** Enforcing a strict frame rate target causes visible banding artifacts due to reduced sampling quality [S1].
*   **Proximity Issues:** Without a target, frame rate drops significantly (e.g., to 40 fps) when the volume is very close to the viewer because more pixels are filled, requiring more rays and higher computational cost [S1].

## Operating characteristics
*   **Time Complexity:** The rendering time is bounded by the allocated frame budget, but the complexity of the volume integral varies with the adaptive sampling distance.
*   **Memory Complexity:** Standard volume rendering memory usage; no significant additional overhead for the adaptation logic itself [S1].
*   **Performance:** Without a VTK fps target, frame rate drops to 40 fps when the volume is very close to the viewer [S1]. With a 300 fps target, frame rate often exceeds 100 fps, maintaining viewer comfort [S1].
*   **Strengths:** Guarantees a minimum frame rate, preventing uncomfortable drops in interactive VR scenarios [S1].
*   **Limitations:** Visual quality degrades (banding) as sampling distance increases to meet time constraints [S1].

## Background
This technique is part of the VTK (Visualization Toolkit) library's support for interactive volume rendering, specifically designed for VR applications where consistent frame rates are critical for user comfort [S1]. It was documented in the context of integrating VTK with Unity for 3D medical imaging visualization [S1].

## Sources
[S1] Virtual interaction and visualisation of 3D medical imaging data with VTK and Unity, https://doi.org/10.1049/htl.2018.5064, fetched 2026-10-28
