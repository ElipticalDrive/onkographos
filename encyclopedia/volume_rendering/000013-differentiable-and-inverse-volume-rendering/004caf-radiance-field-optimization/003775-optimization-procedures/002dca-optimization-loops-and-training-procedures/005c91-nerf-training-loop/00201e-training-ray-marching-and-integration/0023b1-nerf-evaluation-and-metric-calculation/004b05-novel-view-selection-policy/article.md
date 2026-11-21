# Novel View Selection Policy

Also called: Novel View Selection, View Selection based on Uncertainty

## Overview
This policy selects the most informative synthetic views for active learning in neural radiance fields by quantifying information gain through uncertainty metrics [S1, S2]. It distinguishes itself from heuristic baselines (Random, FVS) by using epistemic uncertainty as a proxy for information gain, consistently outperforming them in reconstruction quality [S2]. It builds on [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by providing the mechanism to choose *which* new views to render and evaluate next.

## How it works
The method operates in two phases: initial candidate generation with quality pruning, followed by information-gain-based selection using epistemic uncertainty [S1].

**Inputs:**
*   Pre-trained U-NeRF model (provides RGB, Depth, and associated color/depth uncertainties) [S1].
*   Pre-trained E-SCRNet model (trained on real data $S_{real}$, provides epistemic uncertainty and scene coordinate maps) [S1].
*   Hyperparameters: $z_{min}$ (minimum valid depth), $k$ (number of views to select), sampling rate $r$ (for resolution control) [S1, S2].

**Outputs:**
*   Selected novel view set $S_{high}$ [S1].

**Data Structures:**
*   Rendered Image: Contains RGB tensor, Depth tensor, Color Uncertainty tensor, Depth Uncertainty tensor [S1].
*   Epistemic Uncertainty Map: Per-pixel scalar map indicating model confidence [S1].

**Steps:**

1.  **Generate Initial Candidates:** Render RGB-D images with uncertainty estimation from various viewpoints within the scene using U-NeRF to form the initial synthetic training data set $S_{new}$ [S1].
2.  **Prune by Rendering Quality:** Remove views from $S_{new}$ if any of the following conditions are met [S1]:
    *   Rendered depth $< z_{min}$ (view is too close to scene structure) [S1].
    *   Depth uncertainty is large (indicates incorrect geometry) [S1].
    *   Color uncertainty is large (indicates noisy texture) [S1].
3.  **Compute Epistemic Uncertainty:** For each remaining image in $S_{new}$, apply the pre-trained E-SCRNet to generate an epistemic uncertainty map and a scene coordinate map [S1].
4.  **Filter Unreliable Pixels:** Use the color and depth uncertainties from U-NeRF to mask out unreliable pixels in the epistemic uncertainty map. Pixels with uncertainties exceeding predefined thresholds are ignored [S1].
5.  **Calculate View Score:** Compute the mean epistemic uncertainty of the remaining valid pixels for each image. This mean value serves as the novel view score [S1].
6.  **Select Top-K Views:** Select the top-$k$ images with the highest mean epistemic uncertainty scores to form the final novel view set $S_{high}$ [S1].

**Evaluation Optimization (Sampling Rate):**
To reduce computational cost during candidate evaluation, the new perceptions can be evaluated at a lower resolution controlled by a sampling rate $r$ [S2].
*   $r=1$: Full resolution evaluation.
*   $r>1$: Downsampled evaluation. Performance remains competitive at $r=5$ but degrades gradually when $r>10$ [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_novel_views(u_nerf, e_scrnet, z_min, k, r):
    """
    Selects top-k novel views based on epistemic uncertainty.
    
    Args:
        u_nerf: U-NeRF model instance.
        e_scrnet: E-SCRNet model instance.
        z_min: Minimum valid depth threshold.
        k: Number of views to select.
        r: Sampling rate for evaluation resolution (1 for full).
        
    Returns:
        S_high: List of selected view indices/paths.
    """
    # Step 1: Generate initial candidates
    # Render RGB-D with uncertainties from various viewpoints
    S_new = u_nerf.render_all_viewpoints() 
    
    S_pruned = []
    
    # Step 2: Prune based on rendering quality
    for view in S_new:
        depth = view.depth
        depth_unc = view.depth_uncertainty
        color_unc = view.color_uncertainty
        
        # Check if view is too close
        if depth.min() < z_min:
            continue
            
        # Check for large uncertainties (heuristic thresholds needed)
        # Note: Exact threshold values are not given in sources [S1]
        if depth_unc.mean() > DEPTH_UNC_THRESHOLD:
            continue
        if color_unc.mean() > COLOR_UNC_THRESHOLD:
            continue
            
        S_pruned.append(view)
        
    scores = []
    
    # Step 3-5: Compute score for each pruned view
    for view in S_pruned:
        # Apply E-SCRNet to get epistemic uncertainty map
        # Evaluate at lower resolution if r > 1
        epistemic_map = e_scrnet.predict(view.image_downsampled(r))
        
        # Filter unreliable pixels using U-NeRF uncertainties
        # Mask pixels where uncertainty is too high
        valid_mask = (view.depth_uncertainty_downsampled(r) < DEPTH_UNC_THRESHOLD) & \
                     (view.color_uncertainty_downsampled(r) < COLOR_UNC_THRESHOLD)
        
        # Calculate mean epistemic uncertainty on valid pixels
        if valid_mask.sum() > 0:
            score = epistemic_map[valid_mask].mean()
        else:
            score = 0.0 # Fallback if no valid pixels
            
        scores.append((view, score))
        
    # Step 6: Select top-k
    # Sort by score descending
    scores.sort(key=lambda x: x[1], reverse=True)
    S_high = [item[0] for item in scores[:k]]
    
    return S_high
```

## Subtopics

### Candidate Evaluation and Acquisition Function Comparison
This variant investigates the impact of the acquisition function used for view selection. It compares the proposed uncertainty-based module against heuristic approaches: Random selection and FVS (likely a specific heuristic baseline mentioned in the source) [S2]. The uncertainty-based method (ActiveNeRF Full) consistently outperforms these heuristics in PSNR, SSIM, and LPIPS metrics [S2].

## Operating characteristics
*   **Time Complexity:** Evaluation time depends on the sampling rate $r$. Full resolution ($r=1$) takes 2.20 hours for the ablation setting. Lowering $r$ reduces time slightly (e.g., 2.12h at $r=5$) but degrades quality at high $r$ [S2].
*   **Performance:** In Setting I (20 total observations: 4 initial + 4 extra at 40K, 80K, 120K, 160K iterations), ActiveNeRF Full achieves PSNR 26.24, SSIM 0.856, LPIPS 0.124 [S2].
*   **Comparison:** ActiveNeRF Full outperforms ActiveNeRF+FVS (PSNR 26.05) and ActiveNeRF+Random (PSNR 24.77) [S2].
*   **Strengths:** Reduces computational cost via sampling rate $r$ while maintaining competitive quality at $r=5$ [S2]. Effectively identifies informative views without introducing much computational cost compared to full retraining [S1].
*   **Limitations:** Quality degrades when sampling rate $r > 10$ [S2]. Requires pre-trained E-SCRNet and U-NeRF [S1].

## Background
The method originates from ActiveNeRF, which learns where to see using uncertainty estimation [S2]. It also appears in work leveraging neural radiance fields for uncertainty-aware visual localization, where it is used to boost localization performance by selecting samples with high information gain [S1]. The approach is part of the active learning pipeline for NeRFs, where new views are captured every 40K iterations during training [S2].

## Sources
[S1] Leveraging Neural Radiance Fields for Uncertainty-Aware Visual Localization, https://doi.org/10.1109/icra57147.2024.10610126, fetched 2026-10-26
[S2] ActiveNeRF: Learning Where to See with Uncertainty Estimation, https://link.springer.com/chapter/10.1007/978-3-031-19827-4_14, fetched 2026-10-28
