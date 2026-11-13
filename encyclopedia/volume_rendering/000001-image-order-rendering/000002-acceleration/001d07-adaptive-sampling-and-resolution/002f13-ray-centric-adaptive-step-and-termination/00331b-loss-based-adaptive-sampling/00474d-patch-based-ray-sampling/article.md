# Patch-based Ray Sampling

Also called: Patch-based Rendering

## Overview
A ray sampling strategy where rays are sampled in contiguous 2D patches to form a local image grid, ensuring the spatial adjacency required for local structure analysis such as Discrete Wavelet Transform (DWT) or blur convolution [S1, unofficial][S2]. It is distinguished from random ray sampling, which captures global structures, by focusing on local pixel interference and high-frequency details [S1, unofficial][S2].

## How it works
**Inputs:** Image resolution, patch dimensions, and the specific loss function requiring local adjacency (e.g., $L_{DW}$ or sharpness prior).
**Outputs:** A set of ray origins and directions corresponding to a contiguous block of pixels.

**Data Structures:**
*   **Patch:** A 2D square grid of rays.
*   **Ray Set:** A list of $(x, y)$ pixel coordinates and their corresponding 3D rays.

**Steps:**
1.  **Define Patch Geometry:** Determine the width $W_p$ and height $H_p$ of the square patch.
2.  **Select Patch Location:** Choose a random or deterministic location in the image grid such that the patch fits within the image boundaries.
3.  **Generate Rays:** For each pixel $(u, v)$ in the patch grid ($0 \le u < W_p, 0 \le v < H_p$), compute the ray origin and direction using the camera projection matrix.
4.  **Render:** Trace all rays in the patch to compute color and density values.
5.  **Compute Loss:** Apply the local loss function (e.g., DWT coefficients or blur kernel convolution) on the rendered 2D patch [S1, unofficial][S2].

**Parameters:**
*   **Patch Size:** $192 \times 192$ pixels, requiring 36,864 rays for DWT loss calculation [S1, unofficial].
*   **Sampling Frequency:** DWT loss is calculated every 10 iterations (LLFF) or 150 iterations (NS) [S1, unofficial].

**Edge Cases:**
*   **Boundary Conditions:** Patches near image edges must be handled to avoid out-of-bounds ray generation. [G] Typically, patches are clipped or shifted to remain fully within the image bounds.
*   **Precision:** High-frequency loss functions require accurate ray tracing to avoid aliasing artifacts within the patch [S1, unofficial].

## Subtopics

### Random Patch Sampling
A variant used in deblurring NeRF to model defocus blur interference by rendering a group of rays within a close location [S2]. It reduces training time by allowing neighboring pixels to share rays within a patch [S2].
*   **Definition:** Selects a group of rays within a close location (a patch) to model the interference of neighboring pixels caused by defocus blur [S2].
*   **Formula:** The number of rendered pixels is $(P' + K + 1)^2$, where $P'$ is the width/height of the region to compute intensity and $K$ is the blur kernel size [S2]. The reduction factor for neighboring rays per ray is $K^2 / ((P' + K + 1)^2 / P'^2)$ [S2].
*   **Parameters:** In experiments, $P = 22$ (patch size), $P'^2 = 122$ (number of interesting pixels), and $K = 11$ (blur kernel size) [S2].
*   **When to choose:** Use when dealing with defocus blur and needing to enforce sharpness priors via local neighborhood analysis [S2].

## Operating characteristics
*   **Time Complexity:** Higher per-iteration cost than random sampling due to the large number of rays per patch (e.g., 36,864 rays) [S1, unofficial]. However, random patch sampling can reduce total training time compared to naive full-image blur convolution by sharing neighboring rays [S2].
*   **Memory Complexity:** Requires storing the rendered values for the entire patch simultaneously to compute local transforms [S1, unofficial][S2].
*   **Strengths:** Captures local structures and high-frequency details effectively [S1, unofficial]. Essential for operations requiring spatial adjacency like DWT or blur convolution [S1, unofficial][S2].
*   **Limitations:** Computationally expensive per iteration [S1, unofficial]. Random patch sampling still requires rendering more pixels than the target count due to neighbor inclusion [S2].
*   **When to use:** When the loss function depends on local spatial relationships (e.g., DWT, sharpness priors) [S1, unofficial][S2].
*   **When not:** When global structure is the primary concern (use random sampling instead) [S1, unofficial].

## Background
Originated in the context of Neural Radiance Fields (NeRF) optimization, specifically in DWTNeRF [S1, unofficial] and Sharp-NeRF [S2]. DWTNeRF introduced patch-based sampling to facilitate Discrete Wavelet Transform loss, prioritizing low frequencies early in training [S1, unofficial]. Sharp-NeRF adapted the concept to random patch sampling for deblurring, leveraging sharpness priors [S2].

## Sources
[S1] DWTNeRF: Boosting Few-shot Neural Radiance Fields via Discrete Wavelet Transform, https://arxiv.org/abs/2501.12637v3, fetched 2026-10-27 (unofficial)
[S2] Sharp-NeRF: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior, https://doi.org/10.1109/wacv57701.2024.00367, fetched 2026-10-27
