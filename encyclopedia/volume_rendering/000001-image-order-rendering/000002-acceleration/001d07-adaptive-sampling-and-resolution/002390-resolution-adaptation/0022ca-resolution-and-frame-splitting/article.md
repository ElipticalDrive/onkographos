# Resolution and Frame Splitting

Also called: Performance Optimization: Resolution and Frame Splitting

## Overview
Resolution and Frame Splitting is an image-order volume rendering acceleration technique that reduces computational load by lowering spatial resolution and/or distributing rendering work across multiple frames. It solves the problem of high ray-marching costs in real-time volume rendering, particularly for large-scale effects like volumetric clouds. It distinguishes itself from sibling methods like [Dynamic Resolution Adaptation](2683-dynamic-resolution-adaptation.md) by applying fixed, coarse-grained resolution reductions or temporal splitting strategies rather than fine-grained, metric-driven per-pixel sampling adjustments.

## How it works
**Inputs:** Input buffer (Scene Color RT), camera parameters, hardware capability tier (High/Mid/Low), and target frame rate.
**Outputs:** Final rendered image with volumetric effects composited into the Scene Color RT.

**Data Structures:**
*   **Resolution Buffer:** A render target with dimensions $W' \times H'$, where $W' \le W$ and $H' \le H$.
*   **Frame History Buffer:** Stores previous frame's volumetric contribution for temporal accumulation.

**Steps:**
1.  **Hardware Tier Detection:** Determine if the target platform is High-end or Mid/Low-end.
2.  **Resolution Selection:**
    *   If High-end: Set resolution to $1/2 \times 1/2$ of the original screen resolution ($W' = W/2, H' = H/2$) [S1, unofficial].
    *   If Mid/Low-end: Enable Frame Splitting. This involves rendering only a subset of the volume (e.g., clouds) in the current frame, assuming no intersection with other scene objects to simplify depth handling [S1, unofficial].
3.  **Ray Marching:** Perform volume integration (ray marching) at the selected resolution. For frame splitting, this may involve interlaced or partial coverage strategies similar to those used in *Red Dead Redemption 2* [S1, unofficial].
4.  **Temporal Filtering:** Apply temporal upsampling and filtering to the rendered volumetric buffer. This step combines the current frame's result with history buffers to reduce noise and stabilize the image [S1, unofficial].
5.  **Upsampling:** Upsample the filtered volumetric buffer back to the original screen resolution ($W \times H$). Blue noise dithering may be applied during this step to mitigate aliasing artifacts caused by reduced sampling [S1, unofficial].
6.  **Compositing:** Blend the upsampled volumetric result into the Scene Color Render Target.

**Parameters:**
*   **Resolution Scale:** $0.5$ for High-end [S1, unofficial].
*   **Splitting Factor:** Not explicitly defined, but implies a fraction of the volume or time steps rendered per frame [S1, unofficial].

**Edge Cases:**
*   **Cloud-Object Intersection:** The frame splitting method assumes clouds do not intersect other objects. If they do, additional depth handling is required, which complicates the simple splitting approach [S1, unofficial].
*   **Temporal Stability:** Rapid camera movement or fast cloud motion can cause ghosting artifacts if the temporal history is not properly aligned or weighted [G].

## Subtopics

### [Interlaced Slice Rendering](20997-interlaced-slice-rendering.md)
Interlaced Slice Rendering is an image-order volume rendering optimization that halves the per-frame computational workload by rendering only half of the horizontal slices per frame, alternating between odd and even slices.

## Operating characteristics
*   **Time Complexity:** Reduces ray marching cost by a factor of 4 for $1/2 \times 1/2$ resolution. Frame splitting reduces cost by the splitting factor (e.g., 2x if alternating frames).
*   **Memory Complexity:** Requires additional buffers for history, temporal accumulation, and potentially lower-resolution intermediate targets.
*   **Performance:** Significant FPS improvement on Mid/Low-end hardware. High-end hardware sees moderate improvement due to lower resolution but higher fill-rate requirements for upsampling.
*   **Strengths:** Simple to implement; effective for static or slow-moving scenes; leverages temporal coherence.
*   **Limitations:** Introduces temporal artifacts (ghosting, jitter) if not filtered properly; resolution loss can be noticeable; frame splitting can cause flickering if not handled with care [S1, unofficial].
*   **When to Use:** Real-time rendering of large-scale volumetric effects where performance is critical and visual fidelity can be slightly compromised.
*   **When Not to Use:** Scenes with fast-moving cameras or rapidly changing volumetric structures where temporal stability is hard to maintain.

## Background
This technique is commonly used in real-time game engines for rendering volumetric clouds. It is mentioned in the context of High Dynamic Range Rendering (HDRP) and similar to techniques used in *Red Dead Redemption 2* [S1, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
