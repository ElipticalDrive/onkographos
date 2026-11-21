# Adaptive Pre-integration


## Overview
Adaptive Pre-integration is a technique applied to texture slicing that pre-computes optimal step sizes for each ray based on pre-integrated emission, opacity, and second-order gradients to guarantee error bounds on ray integration [S1, unofficial]. It is an adaptive approach that overcomes pre-integration artifacts by using an adaptive sampling rate based on the actual information stored in the volume and the transfer function used [S2, unofficial]. It builds on [Pre-integrated Volume Rendering](5327-pre-integrated-volume-rendering.md) by dynamically adjusting the discretization of the pre-integration integral rather than using a fixed slab thickness.

## How it works
**Inputs:**
*   Volume dataset.
*   Transfer functions for opacity and color.
*   Camera parameters.

**Outputs:**
*   Final color image.

**Data Structures:**
*   **Importance Volume:** A floating-point texture storing the pre-computed optimal step size for each ray [S1, unofficial].
*   **Ray Traversal Parameters:** Floating-point textures storing current ray positions and accumulated opacity/color [S1, unofficial].
*   **Z-buffer:** Used to track ray termination [S1, unofficial].

**Steps:**
1.  **Initialization:** Render the bounding box of the volume in front-to-back order to provide starting locations for all rays [S1, unofficial]. Initialize ray traversal parameters in floating-point textures [S1, unofficial].
2.  **Step Size Computation:** For each ray, compute the optimal step size. This size depends on the pre-integrated emission and opacity value as well as second-order gradients [S1, unofficial]. The step size is stored in the importance volume [S1, unofficial].
3.  **Ray Traversal:** Update ray positions using the step sizes from the importance volume [S1, unofficial].
4.  **Pre-integration:** Apply pre-integration of opacity and color transfer functions to the volume slabs defined by the entry and exit points and the distance between slices [S1, unofficial].
5.  **Blending (Multi-step Slicing):** Internally blend the results of multiple pre-integrated slabs before writing to the framebuffer to improve image quality [S1, unofficial].
6.  **Termination:** Rays are terminated early or when they leave the volume by setting the z-buffer for the corresponding pixels, avoiding further computation [S1, unofficial].
7.  **Accumulation:** Resampled RGB$\alpha$ textures are accumulated into the frame buffer using back-to-front compositing [S1, unofficial]. If front-to-back compositing is used, accumulated opacities need to be stored for each pixel in the image [S1, unofficial].

**Parameters:**
*   **Over-sampling factor:** Two-times over-sampling has almost the same performance as no over-sampling due to increased cache coherency [S1, unofficial]. Four-times over-sampling yields the best quality-performance tradeoff [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Artifacts:** Standard pre-integration assumes a linear progression of scalar values between slices, which can introduce artifacts [S2, unofficial]. Adaptive sampling mitigates this by adjusting to the actual information in the volume.
*   **Precision:** Ray traversal parameters are stored in floating-point textures, requiring sufficient precision to avoid accumulation errors [S1, unofficial].

## Operating characteristics
*   **Performance:** Two-times over-sampling has almost the same performance as no over-sampling because of increased cache coherency [S1, unofficial].
*   **Quality:** Four-times over-sampling yields the best quality-performance tradeoff using multi-step slicing [S1, unofficial].
*   **Accuracy:** Guarantees error bounds on ray integration by adapting step sizes to local volume properties [S1, unofficial].

## Background
Adaptive Pre-integration was proposed by Roettger et al. [S1, unofficial]. It extends the work of Engel et al. [S1, unofficial] who applied pre-integration of opacity and color transfer functions to texture slicing. Roettger et al. improved image quality by internally blending results of multiple pre-integrated slabs [S1, unofficial].

## Sources
[S1] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
[S2] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
