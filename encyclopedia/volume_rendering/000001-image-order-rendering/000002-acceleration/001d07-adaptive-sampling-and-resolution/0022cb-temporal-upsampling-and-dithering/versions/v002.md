# Temporal Upsampling and Dithering

Also called: Dithering, Temporal Upsampling

## Overview
Temporal Upsampling and Dithering is a rendering technique that combines spatial resolution reduction with temporal accumulation to balance visual fidelity and computational cost. It solves the problem of slicing artifacts caused by insufficient sampling steps in volume rendering by applying blue noise dithering, while using temporal upsampling to reduce the noise introduced by the dithering process. It distinguishes itself from static low-resolution rendering by leveraging frame-to-frame coherence to reconstruct high-resolution details, and from pure dithering by mitigating the resulting visual noise through temporal accumulation.

## How it works
**Inputs and Outputs:**
*   **Input:** Previous frame’s color buffer (history), current frame’s low-resolution color buffer, and camera motion vectors.
*   **Output:** High-resolution color buffer.

**Data Structures:**
*   **History Buffer:** A texture storing accumulated color values, typically at the target resolution.
*   **Motion Vectors:** A texture storing pixel displacement vectors between frames, used for reprojection.

**Steps:**
1.  **Resolution Reduction:** Render the volume at a lower resolution (e.g., $1/2 \times 1/2$) to reduce computational load [S1, unofficial].
2.  **Blue Noise Dithering:** Apply blue noise dithering to the low-resolution samples to break up coherent aliasing patterns and slicing artifacts caused by insufficient sampling steps [S1, unofficial].
3.  **Temporal Accumulation:** Reproject the previous frame’s color buffer into the current frame’s coordinate space using motion vectors. Blend the reprojected history with the current low-resolution, dithered frame.
4.  **Upsampling:** Upsample the accumulated result to the target resolution. This can be done via simple bilinear filtering or more complex reconstruction filters, blending the result into the Scene Color Render Target [S1, unofficial].

**Formulas:**
The accumulated color $C_{acc}$ for a pixel is computed as:
$$ C_{acc} = \alpha C_{current} + (1 - \alpha) C_{reprojected} $$
where $C_{current}$ is the dithered low-resolution color, $C_{reprojected}$ is the reprojected history color, and $\alpha$ is the blending factor (often based on temporal stability or velocity).

**Parameters:**
*   **Dither Pattern:** Blue noise texture.
*   **Resolution Scale:** e.g., 0.5 for half resolution.
*   **Blending Factor:** Typically small (e.g., 0.1-0.2) to allow history to dominate in stable regions.

**Edge Cases and Pitfalls:**
*   **Ghosting:** Caused by incorrect motion vectors or fast-moving objects. Requires robust rejection logic (see [Temporal Anti-Aliasing with Anti-Smear](3614-temporal-anti-anti-smear.md)).
*   **Precision:** Dithering introduces high-frequency noise; insufficient temporal accumulation can leave visible artifacts.

## Subtopics

### [Temporal Anti-Aliasing with Anti-Smear](3614-temporal-anti-aliasing-with-anti-smear.md)
Adds depth-based rejection to prevent smearing artifacts during object movement.

## Operating characteristics
*   **Time Complexity:** Reduced by rendering at lower resolution; temporal accumulation adds overhead for reprojection and blending.
*   **Memory Complexity:** Requires storage for history buffer and motion vectors.
*   **Strengths:** Effective at reducing slicing artifacts and noise while maintaining performance.
*   **Limitations:** Can introduce ghosting or smearing if motion estimation is inaccurate.

## Background
This method is used in real-time volume rendering, particularly for atmospheric effects like clouds, where performance is critical. It builds on the principles of adaptive sampling and resolution [1412-adaptive-sampling-and-resolution.md].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
