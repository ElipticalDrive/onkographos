# Selective Pruning with Sensitivity Ranking

Also called: Final Selection, Single Prune

## Overview
Selective Pruning with Sensitivity Ranking is a post-growth pruning mechanism that selects the final Gaussian population in a single event, matching the cumulative effect of periodic pruning throughout training. It differs from its parent, [Quality Feedback Count Adaptation](10207-quality-feedback-count-adaptation.md), by executing the reduction as a one-time "final selection" rather than a dynamic adjustment based on current learning allowance, and by explicitly ranking Gaussians by sensitivity to determine retention.

## How it works
**Inputs:**
- Current Gaussian population $P$ with count $N_{\text{pre}}$.
- Current training PSNR $\rho$.
- Total training iterations $T_{\text{total}}$.
- Current iteration $T_{\text{curr}}$.
- Sensitivity scores for each Gaussian in $P$.

**Outputs:**
- Pruned Gaussian population $P'$ with count $N_G$.

**Data Structures:**
- Each Gaussian $g_i \in P$ has a scalar sensitivity score $s_i$.
- $s_i$ is defined as the accumulated, opacity-weighted effect on training views.

**Steps:**
1. **Calculate Reference Schedule:** Determine the number of remaining events $h$ in a reference pruning schedule. This schedule spaces pruning events 8% of the total training iterations apart.
   $$ h = \left\lfloor \frac{T_{\text{total}} - T_{\text{curr}}}{0.08 \cdot T_{\text{total}}} \right\rfloor $$
   [G] If $h < 0$, set $h=0$.

2. **Calculate Keep Fraction:** Compute the keep fraction $f$ based on the current PSNR $\rho$ and a multiplier $s > 1$.
   $$ f = \min(1, s \cdot \rho) $$
   [G] The multiplier $s$ compensates for the inability to regrow removed Gaussians. Typical values for $s$ are not specified in the source but must be $>1$.

3. **Calculate Retained Count:** Determine the number of Gaussians to retain $N_G$ using the reference schedule count $h$.
   $$ N_G = N_{\text{pre}} \cdot f^h $$
   [G] If $N_G < 1$, set $N_G = 1$.

4. **Rank and Select:** Rank all Gaussians in $P$ by their sensitivity score $s_i$ in descending order. Select the top $N_G$ Gaussians to form $P'$.

5. **Optimize:** Optimize the retained population $P'$ for the remainder of the training iterations.

**Parameters:**
- $s$: Multiplier for the keep fraction. $s > 1$. Not given in the sources.
- $\rho$: Current training PSNR.
- $T_{\text{total}}$: Total training iterations.
- $T_{\text{curr}}$: Current training iteration.

**Edge Cases and Pitfalls:**
- **No Removal at Low PSNR:** If the current PSNR $\rho$ is at or below a threshold $P^*$, the keep fraction $f=1$, and no Gaussians are removed. [S1, unofficial]
- **Irreversibility:** Gaussians removed during this final selection stage cannot be regrown. Therefore, the multiplier $s > 1$ is necessary to retain a larger fraction than a standard soft prune would at the same PSNR level. [S1, unofficial]
- **Precision:** Ensure $N_G$ is an integer. [G]

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ for sorting Gaussians by sensitivity, where $N$ is the number of Gaussians.
- **Memory Complexity:** $O(N)$ to store sensitivity scores.
- **Strengths:** Provides a deterministic final population size that accounts for the cumulative effect of pruning. The sensitivity ranking ensures that the most "important" Gaussians are retained.
- **Limitations:** The final selection is irreversible. The choice of $s$ and $P^*$ is critical and not specified in the sources.
- **When to use:** At the end of the growth phase, before the final optimization stage.
- **When not to use:** If regrowth is still desired after pruning.

## Background
Selective Pruning with Sensitivity Ranking is implemented in LiteGS (Liao et al., 2025). It is part of a densification strategy that includes growth and periodic pruning. The method aims to compactify the Gaussian representation by selecting the most significant Gaussians in a single event, mimicking the effect of continued periodic pruning.

## Sources
[S1] Gauss What You Need: Compact Gaussian Splatting Across Scene Scales, https://arxiv.org/abs/2609.31248, fetched 2026-10-29 (unofficial)
