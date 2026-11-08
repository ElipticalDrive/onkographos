# Coverage-Aware Pruning

Also called: Effective Opacity-based Pruning, Opacity-based pruning

## Overview
Coverage-Aware Pruning is a density control strategy for 3D Gaussian Splatting that prevents the premature removal of primitives required by specific training presets or camera views. It differs from standard opacity pruning by evaluating the maximum effective opacity across all training configurations, ensuring that Gaussians contributing to any single preset survive even if they are transparent in others [S2, unofficial]. It replaces the opacity reset mechanism used in standard 3DGS [S1].

## How it works
**Inputs:**
- Set of Gaussians $i$.
- Set of training presets $T_{train}$ [S2, unofficial] or set of training cameras $c$ [S1].
- Pruning threshold $\tau_{prune}$ [S1].
- Maximum pruning fraction (e.g., 1% of total Gaussians) [S1].

**Outputs:**
- Binary mask indicating which Gaussians survive pruning.

**Data Structures:**
- Gaussian attributes: center $\boldsymbol{\mu}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, view-dependent attenuation $s_i^{view}$ [S2, unofficial].
- Effective opacity metric $\mathcal{P}_i^c$ [S1] or normalized opacity $\bar{\alpha}_i(T)$ [S2, unofficial].

**Steps:**
1. For each Gaussian $i$, compute the effective opacity metric for each training preset $T \in T_{train}$ or camera $c$.
   - In the preset-based formulation (FactorSplat), compute the normalized opacity $\bar{\alpha}_i(T)$ for each preset [S2, unofficial].
   - In the camera-based formulation (BOGausS), compute $\mathcal{P}_i^c = \mathcal{P}_i^p w_i(p) S$, where $S_i^{(c)}$ is the surface covered by the Gaussian in camera $c$'s image, interpreted as effective opacity [S1].
2. Compute the survival score $\mathcal{P}_{prune}(i)$ as the maximum metric across all presets/cameras:
   $$ \mathcal{P}_{prune}(i) = \max_{T \in T_{train}} \bar{\alpha}_i(T) \quad \text{[S2, unofficial]} $$
   $$ \mathcal{P}_{prune}(i) = \max_c \mathcal{P}_i^c \quad \text{[S1]} $$
3. Compare $\mathcal{P}_{prune}(i)$ against the threshold $\tau_{prune}$.
   - If $\mathcal{P}_{prune}(i) < \tau_{prune}$, the Gaussian is a candidate for removal.
   - Typical threshold: $\tau_{prune} = 0.02$ [S1].
4. Apply a global cap on the number of pruned Gaussians to prevent excessive removal (e.g., limit to 1% of total Gaussians) [S1].
5. Remove candidates that exceed the count limit, prioritizing those with the lowest $\mathcal{P}_{prune}(i)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coverage_aware_pruning(gaussians, presets_or_cameras, threshold=0.02, max_prune_fraction=0.01):
    """
    Prunes Gaussians based on maximum effective opacity across presets/cameras.
    gaussians: list of Gaussian objects with attributes:
        - compute_effective_opacity(preset_or_camera): returns opacity metric
    presets_or_cameras: list of presets or cameras
    """
    survival_scores = []
    for g in gaussians:
        # Compute max effective opacity across all presets/cameras
        max_score = -1.0
        for p in presets_or_cameras:
            score = g.compute_effective_opacity(p)
            if score > max_score:
                max_score = score
        survival_scores.append((g, max_score))

    # Sort by score ascending to find weakest Gaussians
    survival_scores.sort(key=lambda x: x[1])

    # Determine how many to prune
    total_count = len(gaussians)
    max_prune_count = int(total_count * max_prune_fraction)
    
    pruned_count = 0
    candidates_to_prune = []
    
    for g, score in survival_scores:
        if score < threshold:
            candidates_to_prune.append(g)
            pruned_count += 1
            if pruned_count >= max_prune_count:
                break
    
    # Remove candidates
    for g in candidates_to_prune:
        g.mark_for_removal()
```

**Edge Cases and Pitfalls:**
- The score excludes camera-dependent attenuation $s_i^{view}$ [S2, unofficial]. Standard geometric and screen-size pruning criteria remain active and must be applied separately [S2, unofficial].
- Inconsistent results may occur if the metric is not robust across diverse scene types (indoor vs. outdoor) [S1].
- Excessive pruning can degrade quality if the fraction limit is not enforced [S1].

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$ where $N$ is the number of Gaussians and $C$ is the number of presets/cameras.
- **Memory Complexity:** $O(N)$ to store survival scores.
- **Strengths:** Ensures primitives needed by any specific training preset or view are retained, improving stability in multi-preset or multi-view scenarios [S2, unofficial].
- **Limitations:** Requires computing opacity metrics for all presets/cameras, increasing per-step cost compared to single-view pruning [S1][S2, unofficial].
- **When to use:** When training involves multiple appearance presets or diverse camera views where some Gaussians are only visible/active in specific configurations [S2, unofficial].

## Background
Originating from extensions to 3D Gaussian Splatting for multi-preset and optimized rendering, Coverage-Aware Pruning addresses the limitation of standard opacity pruning which may remove Gaussians that are transparent in most views but critical for specific ones [S2, unofficial]. It replaces the opacity reset mechanism in BOGausS [S1] and is used in FactorSplat for appearance-controllable medical volume rendering [S2, unofficial].

## Sources
[S1] BOGausS: Better Optimized Gaussian Splatting, https://doi.org/10.23919/eusipco63237.2025.11226626, fetched 2026-10-26
[S2] FactorSplat: Appearance-Controllable Gaussian Proxies for Medical Volume Rendering, https://doi.org/10.48550/arxiv.2610.02382, fetched 2026-10-26 (unofficial)
