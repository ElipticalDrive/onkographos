# TangoGS

Also called: Gauss What You Need, Capture-Conditioned Learning Allowance

## Overview
TangoGS is a density control strategy for 3D Gaussian Splatting that automatically scales the number of Gaussians to the complexity of the capture without per-scene hyperparameter tuning [S1, unofficial]. It combines a capture-derived "learning allowance" (model sizing based on track-adjusted pixels) with training-based quality feedback (PSNR-driven pruning) to determine the final Gaussian population [S1, unofficial]. It sets itself apart from siblings like [Adaptive Density Control](9188-adaptive-density-control.md) by decoupling the global model budget from the optimization loop, ensuring the model size is invariant to redundant views and scales predictably with capture extent [S1, unofficial].

## How it works
TangoGS operates in three phases: allowance calculation, budgeted densification with soft pruning, and final selection. It uses a shared configuration across all captures, relying on the capture's metadata and real-time training metrics to drive decisions [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Capture metadata: Number of images $N_{\text{img}}$ (including held-out), track-adjusted pixel count $\mathcal{A}$ (derived from sparse model tracks).
    *   Training metrics: Mean per-image PSNR $P$ from the preceding epoch.
    *   Current Gaussian state: Positions, covariances, opacities, colors, and Adam optimizer buffers.
*   **Outputs:**
    *   A pruned and optimized set of Gaussians with a final count $N_G$.

### Data Structures
Gaussians are stored in GPU buffers. To support efficient batch removal, parameters and Adam optimizer buffers are **unclustered, filtered, and reclustered into Morton-ordered groups of 128 elements** [S1, unofficial]. This alignment ensures that removal counts are rounded down to the nearest multiple of 128, preserving group integrity [S1, unofficial].

### Constants and Parameters
*   $g \approx 0.063362106$: Effective Gaussian count scaling factor (Gaussians per track-adjusted pixel) [S1, unofficial].
*   $T_{\text{ref}} = 2,462,476$: Reference count, the rounded geometric mean of linear allowances over 13 standard scenes [S1, unofficial].
*   $s = 1.7$: Scaling factor for final selection keep ratio [S1, unofficial].
*   $c = 0.0075$: Sensitivity coefficient for PSNR feedback [S1, unofficial].
*   $P^* = 20$ dB: Threshold PSNR above which no removal occurs [S1, unofficial].
*   $h$: Number of remaining events in a reference pruning schedule spaced 8% of total iterations apart [S1, unofficial].
*   Split/clone threshold: $0.001$ (maximum gradient magnitude) [S1, unofficial].
*   Opacity decay: Factor $0.5$ per step, floor $1/128$ [S1, unofficial].
*   SH-rest LR divisor: $20$ [S1, unofficial].

### Step-by-Step Method

1.  **Calculate Learning Allowance ($T_{\text{cap}}$):**
    Before training, compute the linear allowance $T_{\text{lin}}$ from the track-adjusted pixel count $\mathcal{A}$:
    $$ T_{\text{lin}} = \text{round} \left[ 0.90 \cdot \text{round}(0.07040234 \cdot \mathcal{A}) \right] $$
    Then compute the capped allowance $T_{\text{cap}}$:
    $$ T_{\text{cap}} = \text{round} \left[ T_{\text{ref}} \cdot \left( \frac{T_{\text{lin}}}{T_{\text{ref}}} \right)^{0.5} \right] $$
    *Note: Nested rounding implements the effective scaling factor $g$ to integer precision [S1, unofficial].*

2.  **Schedule Densification:**
    *   Densification starts at 2% of total training iterations and ends at 80% [S1, unofficial].
    *   Soft pruning begins at ~9% of total iterations [S1, unofficial].
    *   Growth events are spaced **5 epochs** apart for runs $\ge 150$ epochs. For shorter runs, spacing scales proportionally to a minimum of 1 epoch [S1, unofficial].
    *   At each growth event, perform **budgeted densification**: add Gaussians to ramp linearly from the initial point count to the target $T_{\text{adapt}}$ over the growth window [S1, unofficial].

3.  **Compute Quality Feedback ($\rho$):**
    At each growth or pruning event, calculate the mean per-image PSNR $P$ from the just-completed epoch. Compute the keep ratio $\rho$:
    $$ \rho = 0.7382 - 0.0075 \cdot P $$
    *Note: The intercept 0.7382 is derived from $1/s - c(P^* - P^*)$ with rounding, placing the no-removal threshold at ~19.995 dB [S1, unofficial].*

4.  **Soft Pruning (During Growth):**
    *   Rank Gaussians present *before* the current growth event using a **sensitivity score** (accumulated opacity-weighted effect on training views, computed via LiteGS's kernel with unit upstream image multiplier) [S1, unofficial].
    *   Newly created Gaussians are exempt from pruning until they have trained [S1, unofficial].
    *   Keep the top fraction $\rho$ of the ranked Gaussians.
    *   **Guard:** If the removal count would exceed 90% of the population, abort the event to prevent collapse [S1, unofficial].
    *   Removal counts are rounded down to the nearest multiple of 128 [S1, unofficial].

5.  **Final Selection (After Growth Ends):**
    *   Let $N_{\text{pre}}$ be the population entering the final selection event.
    *   Compute the final keep fraction $f$:
        $$ f = \min(1, s \cdot \rho) $$
    *   Calculate the retained count $N_G$:
        $$ N_G = N_{\text{pre}} \cdot f^h $$
    *   Rank all $N_{\text{pre}}$ Gaussians by sensitivity score.
    *   Keep the top $N_G$ Gaussians and optimize them for the remainder of training [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tango_gs_training(total_iterations, initial_gaussians, capture_metadata):
    """
    Simplified high-level flow for TangoGS training.
    capture_metadata contains N_img, track_adjusted_pixels (A), and total_epochs.
    """
    # 1. Pre-compute Allowance
    T_lin = round(0.90 * round(0.07040234 * capture_metadata.A))
    T_ref = 2462476
    T_cap = round(T_ref * (T_lin / T_ref)**0.5)
    
    # 2. Setup Schedule
    densify_start = 0.02 * total_iterations
    densify_end = 0.80 * total_iterations
    prune_start = 0.09 * total_iterations
    
    # Epoch spacing logic
    if total_epochs >= 150:
        epoch_spacing = 5
    else:
        epoch_spacing = max(1, 150 * (total_epochs / 150)) # Proportional scaling
    
    gaussians = initial_gaussians
    current_step = 0
    
    while current_step < total_iterations:
        # Standard 3DGS optimization step
        optimize_step(gaussians, current_step)
        
        # Check for Growth Event
        if densify_start <= current_step <= densify_end:
            if is_growth_event(current_step, epoch_spacing):
                # Calculate target for this window
                # Linear ramp from initial count to T_cap over the growth window
                progress = (current_step - densify_start) / (densify_end - densify_start)
                T_adapt = round(initial_count + progress * (T_cap - initial_count))
                
                # Budgeted Densification: Add Gaussians to reach T_adapt
                gaussians = budgeted_densify(gaussians, T_adapt, threshold=0.001)
                
                # Soft Pruning (if past prune_start)
                if current_step > prune_start:
                    # Compute PSNR from preceding epoch
                    P = compute_mean_per_image_psnr(gaussians)
                    
                    # Compute Keep Ratio
                    rho = 0.7382 - 0.0075 * P
                    
                    # Sensitivity Scoring (LiteGS kernel)
                    scores = compute_sensitivity_scores(gaussians)
                    
                    # Rank and Prune
                    # Only rank Gaussians present BEFORE this growth event
                    eligible = gaussians[:-len(newly_created)] 
                    sorted_indices = argsort_descending(scores[eligible])
                    
                    keep_count = max(1, floor(len(eligible) * rho))
                    
                    # Guard against catastrophic collapse
                    if (len(eligible) - keep_count) / len(eligible) > 0.9:
                        print("Guard abort: removal > 90%")
                    else:
                        # Round removal count to multiple of 128
                        remove_count = len(eligible) - keep_count
                        remove_count = (remove_count // 128) * 128
                        keep_count = len(eligible) - remove_count
                        
                        gaussians = prune_gaussians(gaussians, sorted_indices[:keep_count])
        
        # Check for Final Selection
        if current_step == densify_end:
            N_pre = len(gaussians)
            P = compute_mean_per_image_psnr(gaussians)
            rho = 0.7382 - 0.0075 * P
            s = 1.7
            f = min(1, s * rho)
            
            # h is derived from reference schedule (8% spacing)
            # Assuming h is pre-calculated based on total iterations
            N_G = round(N_pre * (f ** h))
            
            scores = compute_sensitivity_scores(gaussians)
            sorted_indices = argsort_descending(scores)
            gaussians = gaussians[sorted_indices[:N_G]]
            
            # Optimize final set for rest of training
            optimize_final(gaussians, total_iterations - current_step)
            break
            
        current_step += 1
```

## Subtopics

### [Capture-Conditioned Learning Allowance](10214-capture-conditioned-learning-allowance.md)
A method to determine the target Gaussian population size based on capture characteristics (track-adjusted pixels), ensuring model budget invariance to redundant views.

## Operating characteristics
*   **Time Complexity:** Adds minimal overhead to standard 3DGS training. Sensitivity scoring uses LiteGS's kernel, accumulating scores over an epoch without additional renders or gradient modifications [S1, unofficial].
*   **Memory Complexity:** Requires pre-allocated buffers for Morton-ordered groups of 128 elements to support efficient filtering and re-clustering [S1, unofficial].
*   **Performance:** On 13 standard benchmark scenes, TangoGS matches the mean PSNR of the best-performing baseline (LeGS) with 48% fewer Gaussians [S1, unofficial]. On 8 large captures, it automatically scales to larger models without retuning [S1, unofficial].
*   **Strengths:** Single configuration for all 21 tested captures; invariant to view redundancy; robust against catastrophic collapse via guards [S1, unofficial].
*   **Limitations:** Requires accurate track-adjusted pixel counts from a sparse model; relies on PSNR as a proxy for quality, which may not capture all artifacts [S1, unofficial].

## Background
TangoGS was introduced in the paper "Gauss What You Need: Compact Gaussian Splatting Across Scene Scales" [S1, unofficial]. It builds on [Adaptive Density Control](9188-adaptive-density-control.md) and [Score-based Densification](4705-score-based-densification.md) but introduces a capture-derived budget to solve the problem of manual hyperparameter tuning across different scene scales. It is implemented using the LiteGS framework [S1, unofficial].

## Sources
[S1] Gauss What You Need: Compact Gaussian Splatting Across Scene Scales, https://arxiv.org/abs/2609.31248, fetched 2026-10-29 (unofficial)
