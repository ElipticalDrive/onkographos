# Quality Feedback Count Adaptation

Also called: Adaptive Keep Ratio, Keep ratio adaptation, PSNR-Dependent Retention, Sensitivity-Based Pruning

## Overview
Quality Feedback Count Adaptation is a dynamic pruning mechanism that adjusts the final Gaussian count based on the training PSNR, using a keep ratio function to compress or expand the population relative to the learning allowance [S1, unofficial]. It replaces the fixed keep ratio hyperparameter with a dynamic value computed from training progress, controlling the fraction of Gaussians retained during pruning and final selection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Current training PSNR, current Gaussian population size $N$, and two linear map constants calibrated from standard scenes [S1, unofficial].
*   **Output:** A dynamic keep ratio $\rho \in [0, 1]$ and the target number of Gaussians for the next densification step or final selection [S1, unofficial].

**Data Structures:**
*   **Gaussian Population:** A list or array of Gaussian primitives supporting indexing for pruning [S1, unofficial].
*   **Keep Ratio ($\rho$):** A single floating-point value computed per growth/pruning event [S1, unofficial].

**Parameters:**
*   **Linear Map Constants:** Two constants used to map training PSNR to $\rho$. The sources state these are "calibrated once on standard scenes and applied unchanged to large captures" but do not provide the exact numerical values [S1, unofficial].
*   **PSNR Floor:** 20 dB. Below this threshold, the final selection removes nothing [S1, unofficial].
*   **Initial Keep Ratio:** $\rho \approx 0.61$ at the first growth events when training PSNR is below 20 dB [S1, unofficial].

**Steps:**
1.  **Compute Keep Ratio:** At each growth/pruning event, compute the keep ratio $\rho$ from the preceding epoch's training PSNR using a linear map [S1, unofficial]. The keep ratio is a fixed function of training PSNR; a ratio of 0.5 corresponds to 31.8 dB PSNR [S1, unofficial].
2.  **Apply to Densification:** Apply the computed $\rho$ to set the densification target at the growth event [S1, unofficial].
3.  **Apply to Soft Prune:** Apply the computed $\rho$ to set the fraction of Gaussians kept during the soft prune step [S1, unofficial].
4.  **Final Selection:** At the end of training, apply the terminal $\rho$ value to determine the fraction of Gaussians retained from the training population [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_keep_ratio_and_target(psnr_train, a, b, population_size):
    """
    Computes the dynamic keep ratio rho and target Gaussian count.
    The exact linear map constants 'a' and 'b' are not provided in the sources.
    The sources state rho=0.5 corresponds to 31.8 dB PSNR.
    """
    # Guard against PSNR below floor: below 20 dB, the final selection removes nothing.
    # For intermediate steps, rho starts at 0.61 when PSNR < 20 dB.
    if psnr_train < 20.0:
        rho = 0.61
    else:
        # Linear map: rho = a * psnr_train + b
        # Constants 'a' and 'b' must be provided by the caller.
        rho = a * psnr_train + b

    # Calculate target count for densification or pruning
    target_count = int(population_size * rho)
    return rho, target_count
```

**Edge Cases and Pitfalls:**
*   **Terminal Reading Bias:** The terminal keep ratio reading is taken after the last opacity decay, which briefly lowers training PSNR, causing the terminal reading to sit above the last learning reading [S1, unofficial]. This means the final retention rate might be slightly higher than expected based on the final PSNR snapshot [S1, unofficial].
*   **Low PSNR Scenes:** For scenes ending below 27 dB, the keep ratio is high (0.54–0.58), retaining 82–97% of the population [S1, unofficial]. On some large captures (Sci-Art, Residence), the retained fraction $s\rho$ reaches 0.98, meaning the terminal step removes almost nothing and the growth target alone sets the final size [S1, unofficial].
*   **High PSNR Scenes:** For scenes ending above 33 dB, the keep ratio is low (0.46–0.49), retaining 56–64% of the population [S1, unofficial].

## Subtopics

### [Selective Pruning with Sensitivity Ranking](10221-selective-pruning-with-sensitivity-ranking.md)
Selective Pruning with Sensitivity Ranking is a post-growth pruning mechanism that selects the final Gaussian population in a single event, matching the cumulative effect of periodic pruning throughout training. It differs from its parent, [Quality Feedback Count Adaptation](10207-quality-feedback-count-adaptation.md), by executing the reduction as a one-time "final selection" rather than a dynamic adjustment based on current learning allowance, and by explicitly ranking Gaussians by sensitivity to determine retention.

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.

**Measured Performance:**
*   **Standard Captures:** On scenes ending >33 dB (Kitchen, Bonsai, Room, Dr Johnson, Playroom), $\rho$ ranges from 0.46 to 0.49, retaining 56–64% of the population [S1, unofficial].
*   **Medium Captures:** On scenes ending 27–31 dB, $\rho$ ranges from 0.51 to 0.53, retaining 70–82% of the population [S1, unofficial].
*   **Complex/Large Captures:** On scenes ending <27 dB (including 5 of 8 large captures), $\rho$ ranges from 0.54 to 0.58, retaining 82–97% of the population [S1, unofficial].
*   **Adaptive vs. Fixed:** A fixed keep ratio of 0.5 approximates the adaptive rule for high-PSNR captures (max 4% count change, 0.08 dB difference) but selects smaller, lower-quality models for lower-PSNR captures [S1, unofficial]. On the Garden scene, the adaptive ratio of 0.54 retains 1.29× the Gaussians for 0.15 dB higher PSNR and 0.014 lower LPIPS compared to a fixed 0.5 [S1, unofficial]. On complex scenes (Garden, Building, Rubble, Sci-Art), adaptive $\rho$ (0.54–0.58) retains 1.29×–1.66× more Gaussians, yielding 0.13–0.16 dB higher PSNR and 0.006–0.019 lower LPIPS [S1, unofficial].
*   **Training Progress:** During a single run, $\rho$ falls by 0.04 to 0.09 on standard captures and by 0.02 to 0.07 on large captures [S1, unofficial]. Terminal $\rho$ values range from 0.50 to 0.58 across captures [S1, unofficial].

**Scaling:** Not given in the sources.

**Strengths and Limitations:** Not given in the sources.

**When to Use:** Not given in the sources.

**When Not to Use:** Not given in the sources.

## Background
Quality Feedback Count Adaptation is part of the TangoGS method, introduced in "Gauss What You Need: Compact Gaussian Splatting Across Scene Scales" [S1, unofficial]. It is designed to work in conjunction with [Significance-Aware Pruning](9676-significance-aware-pruning.md) and capture-derived learning allowances to achieve compact yet high-quality Gaussian Splatting models across standard and large-scale captures [S1, unofficial]. The method was implemented in TangoGS for densification control and final model selection [S1, unofficial].

## Sources
[S1] Gauss What You Need: Compact Gaussian Splatting Across Scene Scales, https://arxiv.org/abs/2609.31248, fetched 2026-10-29 (unofficial)
