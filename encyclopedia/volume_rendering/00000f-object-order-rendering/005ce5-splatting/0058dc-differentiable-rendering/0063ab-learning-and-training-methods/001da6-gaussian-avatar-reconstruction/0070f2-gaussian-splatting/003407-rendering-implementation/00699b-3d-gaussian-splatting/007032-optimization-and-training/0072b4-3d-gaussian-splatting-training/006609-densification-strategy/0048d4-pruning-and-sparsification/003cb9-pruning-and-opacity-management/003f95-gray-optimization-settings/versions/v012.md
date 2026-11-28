# GRay Optimization Settings

Also called: GRayLQ, GRayHQ, GRay Configuration

## Overview
GRay Optimization Settings define the hyperparameter configurations for training 3D Gaussian primitives using the GRay ray-tracing renderer [S1]. Unlike standard 3D Gaussian Splatting (3DGS) or 3D Gaussian Ray Tracing (3DGRT), these settings explicitly disable opacity resets and size-based pruning, relying instead on opacity thresholds and gradient-based significance metrics to manage the primitive count [S1]. The method offers two distinct presets: GRayLQ for maximum training speed and frame rate, and GRayHQ for higher reconstruction fidelity [S1].

## How it works
The optimization process manages the set of 3D Gaussians $\mathcal{G}$ by applying specific thresholds to opacity and pruning metrics during periodic cleanup steps [S1].

**Inputs:**
*   Set of Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$ with parameters $\{\mathbf{p}_i, \mathbf{s}_i, \mathbf{q}_i, \alpha_i, \mathbf{c}_i\}$ (position, scale, rotation, opacity, spherical harmonic coefficients).
*   Configuration preset (LQ or HQ).

**Data Structures:**
*   Gaussian attributes stored in contiguous arrays for ray traversal.
*   Accumulated opacity $\alpha_i$ and pruning metric $\mu_i$ (significance/weight) per Gaussian.

**Parameters:**
The settings are defined by the following hyperparameters [S1]:
*   $\alpha_{\min}$: Minimum opacity threshold for pruning.
*   $\alpha_{\expo}$: Generalized exponent for opacity blending or metric calculation.
*   $\mu_{\prune}$: Pruning threshold based on accumulated significance.
*   $\eta_{\decay}$: Scale decay factor.
*   $\alpha_{\lazy}$: SH stepping laziness threshold.
*   $\tau_{\min}$: Ray marching step size threshold (used in DHT variants).
*   $\alpha_{\bin}$: Binning threshold for initialization.

**Algorithm Steps:**
1.  **Initialization:** Generate initial Gaussians. Apply binning if enabled. If $\alpha_{\bin} > 0$, remove Gaussians with significance below $\alpha_{\bin}$ [S1].
2.  **Training Loop:** For each iteration:
    a. Render image using ray tracing.
    b. Compute loss and backpropagate.
    c. Update Gaussian parameters using Adam optimizer.
    d. Apply scale decay: $\mathbf{s}_i \leftarrow \mathbf{s}_i \times \eta_{\decay}$ if enabled [S1].
    e. Apply SH laziness: If $\alpha_i < \alpha_{\lazy}$, freeze SH coefficients update [S1].
3.  **Pruning:** Periodically (e.g., every 100–1000 iterations, or as defined by the preset):
    a. Calculate significance metric $\mu_i$ for each Gaussian.
    b. If $\alpha_i < \alpha_{\min}$ OR $\mu_i < \mu_{\prune}$, mark Gaussian for removal [S1].
    c. Remove marked Gaussians and compact buffers [S1].
4.  **No Opacity Reset:** Unlike 3DGS, do not reset opacity of pruned/failed Gaussians [S1].
5.  **No Size Pruning:** Unlike 3DGS, do not prune Gaussians based on their spatial extent/scale [S1].

**Preset Configurations:**

*   **GRayLQ (Low Quality/High Speed):**
    *   $\alpha_{\min} = 0.02$
    *   $\alpha_{\expo} = 3$
    *   $\mu_{\prune} = 5 \times 10^{-7}$
    *   $\eta_{\decay} = 0.9998$
    *   $\alpha_{\lazy} = 12$
    *   Total iterations: 7500
    *   Resolution: Half resolution [S1]

*   **GRayHQ (High Quality):**
    *   $\alpha_{\min} = 0.001$
    *   $\alpha_{\expo} = 1$
    *   $\tau_{\min} = 0.001$
    *   $\mu_{\prune} = 10^{-9}$
    *   Initialization binning: Disabled
    *   Scale decay: Disabled
    *   Total iterations: Standard (implied longer than LQ)
    *   Resolution: Half resolution [S1]

**Edge Cases:**
*   **DHT Truncation:** Disabling the Dynamic Hashing Table (DHT) with high $\tau_{\min}$ causes rapid deterioration. For example, $\tau_{\min}=0.03$ leads to PSNR 24.54 and LPIPS 0.284, while $\tau_{\min}=0.04$ may cause failure to converge due to rapid Gaussian expansion [S1].
*   **Binning:** Setting $\alpha_{\bin} = 0.04$ removes >15% of initial Gaussians with minimal quality loss (PSNR 26.49 $\to$ 26.47) [S1].

## Subtopics

### GRayLQ Preset
A configuration optimized for speed. It uses aggressive pruning ($\alpha_{\min}=0.02$, $\mu_{\prune}=5\times10^{-7}$) and scale decay. It runs for 7500 iterations at half resolution. Results: PSNR 26.23, SSIM 0.839, LPIPS 0.191, Init Time 01:58, Opt Time 00:46, FPS 1051, Init #G 3.27M, Final #G 0.48M [S1].

### GRayHQ Preset
A configuration optimized for quality. It uses conservative pruning ($\alpha_{\min}=0.001$, $\mu_{\prune}=10^{-9}$), disables scale decay and binning, and uses $\alpha_{\expo}=1$. It runs longer at half resolution. Results: PSNR 27.41, SSIM 0.847, LPIPS 0.154, Init Time 01:58, Opt Time 36:11, FPS 55, Init #G 3.97M, Final #G 3.79M [S1].

## Operating characteristics
*   **Speed:** GRayLQ trains in under a minute and achieves >1000 FPS at half resolution, more than doubling 3DGS's frame rate [S1]. GRayHQ takes ~36 minutes for optimization but achieves higher quality [S1].
*   **Quality:** GRayHQ approaches 3DGS-DI quality (PSNR 27.41 vs 3DGS-DI implied higher) but is slower [S1]. GRayLQ has lower quality (PSNR 26.23) [S1].
*   **Memory:** GRayLQ reduces final Gaussian count to 0.48M from 3.27M [S1]. GRayHQ retains 3.79M from 3.97M [S1].
*   **Limitations:** Disabling DHT with high truncation thresholds leads to convergence failure or severe quality drops [S1]. Half resolution is used for these specific benchmarks, potentially limiting direct comparison to full-res methods [S1].

## Background
GRay is a ray-tracing based renderer for 3D Gaussians, introduced in the paper "GRay: Ray Tracing 3D Gaussians Near the Speed of Splats" [S1]. The optimization settings were developed to balance the trade-off between training speed and reconstruction quality, specifically addressing the inefficiencies of opacity resets and size-based pruning found in 3DGS [S1].

## Sources
[S1] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
