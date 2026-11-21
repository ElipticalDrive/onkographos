# GRay Optimization Settings

Also called: GRayLQ, GRayHQ

## Overview
GRay Optimization Settings define the specific hyperparameter configurations used to tune the 3D Gaussian Splatting training loop for the GRay ray-tracing renderer [S1]. They solve the problem of balancing reconstruction quality against training time and final scene complexity by adjusting opacity thresholds, pruning criteria, and spherical harmonic (SH) update strategies [S1]. Unlike standard 3DGS or 3DGRT, GRay settings explicitly disable opacity resets and size-based pruning, relying instead on opacity and significance thresholds to manage Gaussian density [S1].

## How it works
The GRay optimization settings operate within the differentiable training loop described in [Pruning and Opacity Management](9314-pruning-and-opacity-management.md). The method modifies the standard densification and pruning triggers through specific parameter values and behavioral exclusions [S1].

**Inputs and Outputs:**
*   **Inputs:** Training iterations, current Gaussian set $\mathcal{G}$, loss gradients, and preset configuration (LQ or HQ).
*   **Outputs:** Updated Gaussian set with optimized positions, scales, opacities, and SH coefficients.

**Key Behavioral Differences:**
1.  **No Opacity Resets:** Unlike 3DGS and 3DGRT, GRay does not reset opacities of newly split/cloned Gaussians to a low value during densification steps [S1].
2.  **No Size-Based Pruning:** Unlike 3DGS, GRay does not prune Gaussians based on their projected screen size [S1].
3.  **Opacity Thresholding:** Pruning is driven by opacity $\alpha$ and a significance metric $\mu$ [S1].

**Parameters and Formulas:**
The core pruning condition for a Gaussian $i$ is determined by its opacity $\alpha_i$ and a significance score $\mu_i$. The sources define two primary presets with distinct parameter values [S1]:

**GRayLQ (Low Quality/Fast):**
*   $\alpha_{\min} = 0.02$: Minimum opacity threshold.
*   $\alpha_{\expo} = 3$: Generalized exponent for opacity weighting.
*   $\mu_{\prune} = 5 \times 10^{-7}$: Pruning threshold for significance.
*   $\eta_{\decay} = 0.9998$: Scale decay factor.
*   $\alpha_{\lazy} = 12$: SH stepping laziness threshold.
*   Iterations: 7500.

**GRayHQ (High Quality):**
*   $\alpha_{\min} = 0.001$: Minimum opacity threshold.
*   $\alpha_{\expo} = 1$: Generalized exponent (matching 3DGS).
*   $\tau_{\min} = 0.001$: Truncation threshold (matching 3DGRT).
*   $\mu_{\prune} = 10^{-9}$: Pruning threshold.
*   Initialization binning: Disabled.
*   Scale decay: Disabled.

**Implementation Steps:**
1.  **Initialize:** Create initial Gaussian set. If using GRayHQ, disable initialization binning [S1].
2.  **Optimize:** Run differentiable rendering loop for specified iterations [S1].
3.  **Densify:** Trigger densification periodically. Do not reset opacities of new Gaussians [S1].
4.  **Prune:** For each Gaussian $i$:
    *   Calculate significance $\mu_i$ [S1].
    *   If $\alpha_i < \alpha_{\min}$ OR $\mu_i < \mu_{\prune}$, remove Gaussian $i$ [S1].
    *   Do not check projected area/size [S1].
5.  **Update SH:** If $\alpha_i < \alpha_{\lazy}$, update SH coefficients; otherwise, defer update [S1].
6.  **Decay Scales:** Apply scale decay $\eta_{\decay}$ if configured (enabled in LQ, disabled in HQ) [S1].

**Edge Cases:**
*   **DHT Truncation:** Disabling the Differentiable Hash Table (DHT) with high $\tau_{\min}$ causes rapid quality deterioration and potential non-convergence due to Gaussian expansion [S1].
*   **Binning:** Disabling initialization binning increases initial Gaussian count (e.g., 3.27M to 3.97M) with minimal quality gain in some configurations [S1].

## Subtopics

### GRayLQ Preset
A configuration optimized for speed and low final Gaussian count. It uses aggressive opacity ($\alpha_{\min}=0.02$) and significance ($\mu_{\prune}=5\times10^{-7}$) thresholds, along with scale decay ($\eta_{\decay}=0.9998$) and SH laziness ($\alpha_{\lazy}=12$) [S1]. It runs for 7500 iterations at half resolution [S1]. Results: PSNR 26.23, SSIM 0.839, LPIPS 0.191, Init Time 01:58, Opt Time 00:46, FPS 1051, Init #G 3.27M, Final #G 0.48M [S1].

### GRayHQ Preset
A configuration optimized for reconstruction quality. It uses conservative opacity ($\alpha_{\min}=0.001$) and significance ($\mu_{\prune}=10^{-9}$) thresholds, disables scale decay, and disables initialization binning [S1]. It runs at half resolution [S1]. Results: PSNR 27.41, SSIM 0.847, LPIPS 0.154, Init Time 01:58, Opt Time 36:11, FPS 55, Init #G 3.97M, Final #G 3.79M [S1].

## Operating characteristics
*   **Time Complexity:** GRayLQ trains in under a minute (00:46 optimization time), more than doubling 3DGS's frame rate [S1]. GRayHQ takes significantly longer (36:11 optimization time) but improves quality over GRayLQ [S1].
*   **Memory Complexity:** GRayLQ reduces final Gaussian count to 0.48M from 3.27M [S1]. GRayHQ retains 3.79M Gaussians from 3.97M [S1].
*   **Performance:** GRayLQ achieves FPS 1051 at half resolution [S1]. GRayHQ achieves FPS 55 at half resolution [S1].
*   **Strengths:** GRayLQ offers extremely fast training and high rendering FPS [S1]. GRayHQ approaches 3DGS-DI quality [S1].
*   **Limitations:** GRayHQ is slow to train [S1]. Disabling DHT with high thresholds causes convergence failure [S1].

## Background
GRay Optimization Settings originate from the GRay paper, which introduces a ray-tracing approach for 3D Gaussians [S1]. The settings were developed to balance the trade-off between training speed and reconstruction quality, specifically tailored for the GRay renderer's capabilities [S1].

## Sources
[S1] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
