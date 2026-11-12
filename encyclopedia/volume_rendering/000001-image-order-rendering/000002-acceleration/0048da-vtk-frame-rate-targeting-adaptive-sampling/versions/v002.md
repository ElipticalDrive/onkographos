# VTK Frame Rate Targeting & Adaptive Sampling

Also called: AutoAdjustSampleDistances, SetDesiredUpdateRate

## Overview
This method maintains a target frame rate in VTK volume rendering by dynamically adjusting the sampling distance per frame based on the allocated render time budget [S1]. It sets apart from static adaptive methods by using a global temporal constraint (frame rate) rather than a local spatial metric (gradient/error) to modulate sampling density [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Desired frame rate target ($F_{target}$), scene geometry (props), volume data.
*   **Output:** Adjusted sampling distance for each volume mapper, resulting in a rendered image at or above $F_{target}$.

**Data Structures:**
*   `vtkExternalOpenGLRenderWindow`: Manages the render loop and time budgeting.
*   `vtkOpenGLGPUVolumeRayCastMapper`: Contains the `AutoAdjustSampleDistances` flag and the current sampling distance parameter.

**Steps:**
1.  **Set Target:** Call `SetDesiredUpdateRate(F_{target})` on the `vtkExternalOpenGLRenderWindow` instance [S1].
2.  **Calculate Budget:** VTK calculates the per-frame time budget $T_{budget} = 1 / F_{target}$ [S1].
3.  **Allocate Time:** The total $T_{budget}$ is divided among all props in the scene. If $N$ props are present, each prop receives an approximate time allocation $T_{prop} \approx T_{budget} / N$ [S1].
4.  **Adapt Sampling:** For each `vtkOpenGLGPUVolumeRayCastMapper` with `AutoAdjustSampleDistances` enabled:
    *   The mapper monitors its actual rendering time.
    *   If the rendering time exceeds $T_{prop}$, the mapper increases the sampling distance (reducing sample count).
    *   If the rendering time is well below $T_{prop}$, the mapper may decrease the sampling distance (increasing sample count) [S1].
5.  **Render:** Execute the volume ray casting with the adjusted sampling distance [S1].

**Parameters:**
*   $F_{target}$: The desired frame rate for the VTK render window. In VR applications with 3 views (left eye, right eye, screen preview) and a 90 fps application target, $F_{target}$ is often set to 300 fps to account for overhead and multiple viewports [S1].
*   `AutoAdjustSampleDistances`: Boolean flag on `vtkOpenGLGPUVolumeRayCastMapper`. Must be enabled for the adaptation to occur [S1].

**Edge Cases and Pitfalls:**
*   **Banding Artifacts:** Reducing sampling distance to meet frame rate targets causes visible banding artifacts due to reduced sampling quality [S1].
*   **Close Proximity:** Without a target, frame rates drop significantly (e.g., to 40 fps) when the volume is close to the viewer because more pixels are filled, requiring more rays [S1].
*   **Overhead:** The target must account for rendering multiple views (e.g., stereo + preview) and system overhead; setting it too low may cause instability, while setting it too high may not provide the desired quality [S1].

## Operating characteristics
*   **Time Complexity:** The adaptation logic adds negligible overhead to the render loop; the primary cost is the volume rendering itself, which is reduced by increasing sampling distance [S1].
*   **Memory Complexity:** No additional memory structures are required beyond standard VTK render state [S1].
*   **Performance:** With a 300 fps target, frame rates often exceed 100 fps, ensuring viewer comfort in VR [S1]. Without a target, frame rates can drop to 40-50 fps for CT data at close range (0.5m) [S1].
*   **Strengths:** Guarantees a minimum frame rate, preventing uncomfortable latency in interactive VR applications [S1].
*   **Limitations:** Image quality degrades (banding) as sampling distance increases to meet time constraints [S1].

## Background
This technique is part of the Visualization Toolkit (VTK) library, specifically within its OpenGL rendering backend. It was documented in research regarding virtual interaction and visualization of 3D medical imaging data using VTK and Unity [S1]. The method addresses the challenge of maintaining interactive frame rates in VR when rendering complex volume data that varies in computational cost based on camera distance and scene complexity [S1].

## Sources

[S1] Virtual interaction and visualisation of 3D medical imaging data with VTK and Unity, https://doi.org/10.1049/htl.2018.5064, fetched 2026-10-28
