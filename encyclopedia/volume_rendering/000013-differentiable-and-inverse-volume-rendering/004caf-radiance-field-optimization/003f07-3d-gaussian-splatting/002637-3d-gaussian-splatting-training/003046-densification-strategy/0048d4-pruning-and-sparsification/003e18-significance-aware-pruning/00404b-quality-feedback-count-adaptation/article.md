# Quality Feedback Count Adaptation

Also called: Adaptive Keep Ratio, Keep ratio adaptation, PSNR-Dependent Retention, Sensitivity-Based Pruning

## Overview
Quality Feedback Count Adaptation is a dynamic pruning mechanism that adjusts the final Gaussian count based on the training PSNR ($\rho$), using a keep ratio function to compress or expand the population relative to the learning allowance [S1, unofficial]. It replaces the fixed keep ratio hyperparameter with a dynamic value computed from training progress, controlling the fraction of Gaussians retained during pruning and final selection [S1, unofficial]. This method builds on [Significance-Aware Pruning](9676-significance-aware-pruning.md) by linking the pruning intensity directly to the reconstruction quality achieved so far, ensuring that complex scenes retain more Gaussians while simple scenes are compressed more aggressively.

## How it works
**Inputs and Outputs:**
*   **Input:** Current training PSNR ($\text{PSNR}_{\text{train}}$), current Gaussian population size $N$, and two linear map constants ($a, b$) calibrated from standard scenes [S1, unofficial].
*   **Output:** A dynamic keep ratio $\rho \in [0, 1]$ and the target number of Gaussians for the next densification step or final selection [S1, unofficial].

**Data Structures:**
*   **Gaussian Population:** A list or array of Gaussian primitives. No specific bit-packing is required for the ratio calculation itself, but the population must support indexing for pruning [S1, unofficial].
*   **Keep Ratio ($\rho$):** A single floating-point value computed per growth/pruning event [S1, unofficial].

**Parameters:**
*   **Linear Map Constants ($a, b$):** Two constants used to map training PSNR to $\rho$. The sources state these are "calibrated once on standard scenes and applied unchanged to large captures" but do not provide the exact numerical values [S1, unofficial].
*   **PSNR Floor:** 20 dB. Below this threshold, the final selection removes nothing [S1, unofficial].
*   **Initial Keep Ratio:** $\rho \approx 0.61$ at the first growth events (when training PSNR is below 20 dB) [S1, unofficial].

**Steps:**
1.  **Compute Keep Ratio:** At each growth/pruning event, compute the keep ratio $\rho$ from the preceding epoch's training PSNR using a linear map [S1, unofficial]. The sources describe $\rho$ as a "fixed function of training PSNR" and provide a reference point: a ratio of 0.5 corresponds to 31.8 dB PSNR [S1, unofficial]. The exact formula is not explicitly given, but it is described as a linear map with two constants [S1, unofficial].
2.  **Apply to Densification:** Apply the computed $\rho$ to set the densification target at the growth event [S1, unofficial].
3.  **Apply to Soft Prune:** Apply the computed $\rho$ to set the fraction of Gaussians kept during the soft prune step [S1, unofficial].
4.  **Final Selection:** At the end of training, apply the terminal $\rho$ value to determine the fraction of Gaussians retained from the training population [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_keep_ratio(psnr_train, a, b, psnr_floor=20.0):
    """
    Computes the dynamic keep ratio rho based on training PSNR.
    Note: The exact linear map constants 'a' and 'b' are not provided in the sources.
    The function assumes a linear relationship: rho = a * psnr + b.
    The sources state rho=0.5 corresponds to 31.8 dB.
    """
    # Guard against PSNR below floor
    if psnr_train < psnr_floor:
        # Below 20 dB, the final selection removes nothing.
        # For intermediate steps, rho starts at 0.61 when PSNR < 20 dB.
        return 0.61
    
    # Linear map: rho = a * psnr + b
    # Since 'a' and 'b' are not given, we cannot compute the exact value.
    # However, we know rho decreases as PSNR increases (from 0.61 to ~0.46).
    # This implies 'a' is negative.
    rho = a * psnr_train + b
    
    # Clamp rho to a reasonable range if necessary, though the sources
    # imply the linear map handles the range naturally.
    return rho

def apply_pruning(gaussians, rho):
    """
    Prunes the Gaussian population based on the keep ratio rho.
    Assumes 'gaussians' is sorted by significance (as per parent method).
    """
    n_total = len(gaussians)
    n_keep = int(n_total * rho)
    
    # Retain the top n_keep Gaussians
    # The parent method [Significance-Aware Pruning] handles the significance ranking.
    # Here we just select the top fraction.
    retained_gaussians = gaussians[:n_keep]
    
    return retained_gaussians

def quality_feedback_count_adaptation(gaussians, psnr_train, a, b):
    """
    Main adaptation step.
    """
    # 1. Compute keep ratio
    rho = compute_keep_ratio(psnr_train, a, b)
    
    # 2. Apply to pruning
    new_gaussians = apply_pruning(gaussians, rho)
    
    return new_gaussians, rho
```

**Edge Cases and Pitfalls:**
*   **Terminal Reading Bias:** The terminal keep ratio reading is taken after the last opacity decay, which briefly lowers training PSNR, causing the terminal reading to sit above the last learning reading [S1, unofficial]. This means the final retention rate might be slightly higher than expected based on the final PSNR snapshot.
*   **Low PSNR Scenes:** For scenes ending below 27 dB, the keep ratio is high (0.54–0.58), retaining 82–97% of the population [S1, unofficial]. On some large captures (Sci-Art, Residence), the retained fraction $s\rho$ reaches 0.98, meaning the terminal step removes almost nothing and the growth target alone sets the final size [S1, unofficial].
*   **High PSNR Scenes:** For scenes ending above 33 dB, the keep ratio is low (0.46–0.49), retaining 56–64% of the population [S1, unofficial].

## Subtopics

### [Selective Pruning with Sensitivity Ranking](10221-selective-pruning-with-sensitivity-ranking.md)
Selective Pruning with Sensitivity Ranking is a post-growth pruning mechanism that selects the final Gaussian population in a single event, matching the cumulative effect of periodic pruning throughout training. It differs from its parent, [Quality Feedback Count Adaptation](10207-quality-feedback-count-adaptation.md), by executing the reduction as a one-time "final selection" rather than a dynamic adjustment based on current learning allowance, and by explicitly ranking Gaussians by sensitivity to determine retention.

## Operating characteristics
**Time and Memory Complexity:**
*   The computation of $\rho$ is $O(1)$ given the PSNR.
*   Pruning is $O(N)$ where $N$ is the number of Gaussians, assuming the population is already sorted by significance (as per the parent method). If sorting is required, it is $O(N \log N)$.
*   Memory complexity is $O(N)$ to store the Gaussian population.

**Measured Performance:**
*   **Standard Captures:** On scenes ending >33 dB (Kitchen, Bonsai, Room, Dr Johnson, Playroom), $\rho$ ranges from 0.46 to 0.49, retaining 56–64% of the population [S1, unofficial].
*   **Medium Captures:** On scenes ending 27–31 dB, $\rho$ ranges from 0.51 to 0.53, retaining 70–82% of the population [S1, unofficial].
*   **Complex/Large Captures:** On scenes ending <27 dB (including 5 of 8 large captures), $\rho$ ranges from 0.54 to 0.58, retaining 82–97% of the population [S1, unofficial].
*   **Adaptive vs. Fixed:** A fixed keep ratio of 0.5 approximates the adaptive rule for high-PSNR captures (max 4% count change, 0.08 dB difference) but selects smaller, lower-quality models for lower-PSNR captures [S1, unofficial]. For example, on the Garden scene, the adaptive ratio of 0.54 retains 1.29× the Gaussians for 0.15 dB higher PSNR and 0.014 lower LPIPS compared to a fixed 0.5 [S1, unofficial]. On complex scenes (Garden, Building, Rubble, Sci-Art), adaptive $\rho$ (0.54–0.58) retains 1.29×–1.66× more Gaussians, yielding 0.13–0.16 dB higher PSNR and 0.006–0.019 lower LPIPS [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Automatically adjusts model complexity to scene difficulty, improving quality on complex scenes without manual tuning. Reduces model size on simple scenes.
*   **Limitations:** Requires accurate PSNR estimation during training. The linear map constants are scene-dependent and not provided in the sources, requiring calibration. The terminal reading bias can lead to slightly higher retention than expected.

**When to Use:**
*   Use when training 3D Gaussian Splatting models across diverse scene complexities and you want to automate the trade-off between model size and reconstruction quality.
*   Particularly effective for large-scale captures where a fixed keep ratio might underfit complex details.

**When Not to Use:**
*   If computational resources are extremely limited and you need a fixed, predictable model size regardless of quality.
*   If you cannot estimate PSNR during training (e.g., in real-time applications without a validation set).

## Background
Quality Feedback Count Adaptation is part of the TangoGS method, introduced in "Gauss What You Need: Compact Gaussian Splatting Across Scene Scales" [S1, unofficial]. It is designed to work in conjunction with [Significance-Aware Pruning](9676-significance-aware-pruning.md) and capture-derived learning allowances to achieve compact yet high-quality Gaussian Splatting models across standard and large-scale captures [S1, unofficial]. The method was implemented in TangoGS for densification control and final model selection [S1, unofficial].

## Sources
[S1] Gauss What You Need: Compact Gaussian Splatting Across Scene Scales, https://arxiv.org/abs/2609.31248, fetched 2026-10-29 (unofficial)
