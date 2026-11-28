# NeRF-GT Baseline Training Setup

Also called: NeRF-GT, HDR-NeRF Baseline

## Overview
The NeRF-GT baseline establishes a reference training protocol for Neural Radiance Fields (NeRF) using High Dynamic Range (HDR) inputs, serving as an upper-bound comparison for methods designed to synthesize HDR views from Low Dynamic Range (LDR) inputs [S1]. It differs from standard NeRF implementations by explicitly addressing the dynamic range mismatch between HDR scene representations and LDR ground truth observations, utilizing a tone-mapping strategy during loss computation [S1]. Unlike the standard [NeRF Training Setup](13271-nerf-training-setup.md) which typically assumes consistent exposure, this setup handles the high dynamic range challenges that can lead to poor encoding of scene areas if not managed correctly [S1].

## How it works
The method extends the standard differentiable volume rendering pipeline defined in [NeRF Training Loop](1522-training-loop-setup.md) by modifying the loss computation stage to accommodate HDR-to-LDR comparisons.

**Inputs and Outputs:**
*   **Input:** A set of camera poses and corresponding HDR images (or consistently exposed LDR images) representing the scene.
*   **Output:** A trained NeRF model (MLP) capable of rendering novel views.

**Data Structures:**
*   **NeRF Model:** Standard MLP taking 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ as input, outputting volume density $\sigma$ and color $\mathbf{c}$ [S1].
*   **Ray Samples:** $N$ points sampled along each ray, with parameters defined below.

**Parameters:**
*   **Coarse samples ($N_c$):** 64 samples per ray [S2, unofficial].
*   **Fine samples ($N_f$):** 128 samples per ray [S2, unofficial].
*   **Batch size:** 2,048 rays per batch [S2, unofficial].
*   **Resolution:** 800 × 600 pixels [S2, unofficial].
*   **Max iterations:** 200,000 solver steps [S2, unofficial].

**Steps:**
1.  **Ray Sampling:** Generate rays from the camera poses and sample points along each ray within the scene bounds using the coarse and fine sampling strategies [S2, unofficial].
2.  **Network Query:** Evaluate the NeRF MLP at each sampled point to obtain density $\sigma_i$ and color $\mathbf{c}_i$ [S1].
3.  **Volume Rendering:** Compute the predicted HDR pixel color $\hat{\mathbf{C}}_{HDR}(\mathbf{r})$ for each ray using the discretized volume rendering equation:
    $$ \hat{\mathbf{C}}_{HDR}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S1].
4.  **Tone Mapping:** Apply a tone-mapping operator $f_{TM}(\cdot)$ to the predicted HDR pixel values to convert them into LDR space:
    $$ \hat{\mathbf{C}}_{LDR} = f_{TM}(\hat{\mathbf{C}}_{HDR}) $$
    [S1].
5.  **Loss Computation:** Compute the photometric loss (typically Mean Squared Error) between the tone-mapped predicted LDR values and the tone-mapped ground truth LDR views:
    $$ \mathcal{L} = \| \hat{\mathbf{C}}_{LDR} - \mathbf{C}_{GT, LDR} \|^2 $$
    [S1].
6.  **Optimization:** Update the NeRF model parameters using gradient descent [S1].

**Pitfalls:**
*   Training directly on HDR views can fail to encode all areas of a scene well due to the high dynamic range, even when tone-mapped colors are used for loss calculation [S1]. This is a critical limitation of the baseline compared to methods that explicitly model exposure or use LDR inputs directly [S1].

## Subtopics

### NeRF Baseline
The standard NeRF baseline serves as the foundational comparison for the NeRF-GT setup. It typically uses LDR inputs with consistent exposure and does not incorporate the HDR-specific tone-mapping loss strategy described in the NeRF-GT baseline. It is chosen when the scene does not exhibit high dynamic range artifacts or when HDR synthesis is not the primary goal [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The training time is approximately 0.40 seconds per iteration [S2, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Rendering time is approximately 32 seconds per frame [S2, unofficial].
*   **Scaling:** The method processes 2,048 rays per batch at 800 × 600 resolution [S2, unofficial].
*   **Strengths:** Provides an upper-bound performance metric for HDR synthesis tasks by training directly on HDR data [S1].
*   **Limitations:** Struggles to encode all scene areas well due to the high dynamic range, leading to potential quality degradation in tone-mapped outputs [S1].
*   **When to use:** When establishing a performance baseline for HDR NeRF methods or when HDR ground truth data is available and high dynamic range fidelity is required [S1].
*   **When not to use:** When computational resources are limited and LDR inputs are the only available data, or when the high dynamic range causes encoding failures [S1].

## Background
The NeRF-GT baseline originates from the work on HDR-NeRF, which introduced the concept of training NeRF models directly from HDR views to synthesize high dynamic range novel views [S1]. The training configuration parameters (64 coarse samples, 128 fine samples, 200,000 iterations) are adapted from the original NeRF paper's recommended settings for the DTU dataset, as cited in comparative studies [S2, unofficial]. The method highlights the challenges of applying standard NeRF pipelines to HDR data, motivating the development of more robust HDR synthesis techniques [S1].

## Sources
[S1] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
[S2] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
