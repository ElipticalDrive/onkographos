# GRay Optimization Settings

Also called: GRayLQ, GRayHQ

## Overview
GRay Optimization Settings define the specific hyperparameters and procedural modifications used in the GRay ray-tracing framework to balance training speed, memory usage, and reconstruction quality [S1]. Unlike standard 3D Gaussian Splatting (3DGS) or 3DGRT, GRay disables opacity resets and size-based pruning, relying instead on strict opacity and gradient-based thresholds to manage primitive counts [S1]. It offers two distinct presets: GRayLQ for maximum speed and efficiency, and GRayHQ for high-fidelity reconstruction [S1].

## How it works
The optimization process modifies the standard 3DGS training loop by adjusting pruning thresholds, decay rates, and initialization strategies. The method relies on the shared principles of [Pruning and Opacity Management](9314-pruning-and-opacity-management.md), specifically opacity-based removal and periodic execution, but diverges in specific mechanics.

### Inputs and Outputs
*   **Inputs:** Initial set of 3D Gaussians, camera poses, input images, and preset configuration (LQ or HQ).
*   **Outputs:** Optimized Gaussian attributes (position, scale, rotation, SH coefficients, opacity) and final primitive count.

### Data Structures
The implementation maintains the standard 3DGS attribute buffers. Key modifications involve the handling of opacity $\alpha$ and scale $s$.
*   **Opacity $\alpha$:** Clamped to $[\alpha_{\min}, 1]$ during updates. No reset to a small value is performed [S1].
*   **Scale $s$:** Not used for pruning decisions [S1]. Subject to exponential decay in GRayLQ [S1].

### Parameters and Configuration
Two presets are defined [S1]. All values are taken directly from the source [S1].

**GRayLQ Preset (Low Quality/High Speed):**
*   $\alpha_{\min} = 0.02$
*   $\alpha_{\text{expo}} = 3$ (Generalized exponent for opacity weighting)
*   $\mu_{\text{prune}} = 5 \times 10^{-7}$ (Gradient-based pruning threshold)
*   $\eta_{\text{decay}} = 0.9998$ (Scale decay factor)
*   $\alpha_{\text{lazy}} = 12$ (SH stepping laziness threshold)
*   Total Iterations: 7500
*   Resolution: Half resolution (for reported metrics)

**GRayHQ Preset (High Quality):**
*   $\alpha_{\min} = 0.001$
*   $\alpha_{\text{expo}} = 1$ (Matches 3DGS)
*   $\tau_{\min} = 0.001$ (Ray marching step size threshold, matches 3DGRT)
*   $\mu_{\text{prune}} = 10^{-9}$
*   Initialization binning: Disabled
*   Scale decay: Disabled entirely
*   Total Iterations: Standard (implied >7500 based on Opt Time)
*   Resolution: Half resolution (for reported metrics)

### Algorithm Steps
1.  **Initialization:** Generate initial Gaussians. If using GRayHQ, disable initialization binning [S1]. If using GRayLQ, apply binning with threshold $\tau_{\text{bin}} = 0.04$ to remove >15% of initial Gaussians, trading minimal quality for reduced count [S1].
2.  **Training Loop:** For each iteration $i$ from 1 to $N_{\text{iter}}$:
    a. **Rendering:** Render image using ray tracing. Use $\tau_{\min}$ to control ray marching precision [S1].
    b. **Loss Computation:** Compute loss against ground truth.
    c. **Backpropagation:** Compute gradients for all Gaussian attributes [S1].
    d. **Optimization Step:** Update attributes using Adam optimizer [S1].
    e. **Scale Decay (GRayLQ only):** Apply exponential decay to scales: $s_{new} = s_{old} \times \eta_{\text{decay}}$ [S1].
    f. **SH Laziness (GRayLQ only):** If opacity $\alpha < \alpha_{\text{lazy}}$, freeze SH coefficient updates to save computation [S1].
    g. **Pruning:** Periodically (every $n$ iterations, typically 100-500 in standard 3DGS, though exact period for GRay is "Not given in the sources"), remove Gaussians based on:
        i. Opacity: If $\alpha < \alpha_{\min}$, remove Gaussian [S1].
        ii. Gradient Norm: If gradient norm $< \mu_{\text{prune}}$, remove Gaussian [S1].
        iii. **Note:** Do NOT prune based on Gaussian size/scale [S1].
        iv. **Note:** Do NOT reset opacity of surviving Gaussians [S1].
3.  **Termination:** After $N_{\text{iter}}$, output final Gaussians.

### Edge Cases and Pitfalls
*   **DHT Truncation:** Disabling the Differentiable Hash Table (DHT) or using high $\tau_{\min}$ values causes rapid quality deterioration and potential divergence due to uncontrolled Gaussian expansion [S1].
*   **Precision:** GRayLQ is designed for half-resolution training to manage memory; full-resolution performance metrics are not provided in the primary comparison table [S1].
*   **Convergence:** High pruning thresholds (e.g., $\tau_{\min} > 0.03$ in NoDHT variants) can cause catastrophic failure to converge [S1].

## Subtopics

### GRayLQ Preset
A configuration optimized for speed. It uses aggressive pruning ($\alpha_{\min}=0.02$, $\mu_{\text{prune}}=5\times10^{-7}$), scale decay, and SH laziness to reduce training time to under a minute at half resolution [S1]. Choose this when inference speed and low training cost are prioritized over peak PSNR/SSIM [S1].

### GRayHQ Preset
A configuration optimized for quality. It uses minimal pruning ($\alpha_{\min}=0.001$, $\mu_{\text{prune}}=10^{-9}$), disables scale decay, and disables initialization binning to preserve geometric detail [S1]. Choose this when high fidelity is required and training time is less critical [S1].

## Operating characteristics
*   **Time Complexity:** GRayLQ trains in under a minute (Opt Time 00:46) at half resolution, more than doubling 3DGS's frame rate [S1]. GRayHQ takes significantly longer (Opt Time 36:11) due to finer pruning and lack of decay optimizations [S1].
*   **Memory Complexity:** GRayLQ reduces final Gaussian count to 0.48M from 3.27M init [S1]. GRayHQ retains 3.79M from 3.97M init [S1].
*   **Quality (Half Resolution):**
    *   GRayLQ: PSNR 26.23, SSIM 0.839, LPIPS 0.191 [S1].
    *   GRayHQ: PSNR 27.41, SSIM 0.847, LPIPS 0.154 [S1].
*   **Strengths:** Extremely fast training for LQ variant; high FPS (1051 for LQ, 55 for HQ) [S1].
*   **Limitations:** HQ variant is slower than standard 3DGS; LQ variant has lower PSNR/SSIM than HQ [S1].
*   **When to use:** Use GRayLQ for real-time applications or rapid prototyping. Use GRayHQ for offline high-quality rendering where 3DGS-DI quality is approached but with ray-tracing benefits [S1].

## Background
GRay is a ray-tracing based 3D Gaussian Splatting method introduced in the paper "GRay: Ray Tracing 3D Gaussians Near the Speed of Splats" [S1]. The optimization settings were developed to leverage the efficiency of ray tracing while maintaining competitive quality metrics, specifically addressing the trade-offs between training speed and reconstruction fidelity through distinct presets [S1].

## Sources
[S1] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
