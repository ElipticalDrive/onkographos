# Moderate Pruning

Also called: Contribution Pruning, Contribution-Prune, Multi-view loss-driven pruning

## Overview
Moderate Pruning is a strategy for removing redundant or artifact-inducing 3D Gaussians by combining opacity thresholds with a multi-view, loss-driven contribution score [S2]. It solves the problem of aggressive performance degradation caused by standard opacity-only pruning, which often removes essential geometry [S2]. Unlike its sibling [Adaptive Opacity Pruning](15576-adaptive-opacity-pruning.md), which adjusts opacity thresholds dynamically, Moderate Pruning introduces a secondary metric based on training loss to distinguish between truly non-contributing Gaussians and those that are merely occluded or low-opacity but geometrically significant [S1, unofficial][S2].

## How it works
**Inputs:**
- List of $N$ 3D Gaussians, each with opacity $o_i$ and parameters defining its position, scale, rotation, and spherical harmonics.
- A queue of training views processed during optimization.
- A current training iteration state.

**Outputs:**
- A binary pruning mask $\mathcal{M}_{prune}$ indicating which Gaussians to remove.

**Data Structures:**
- **Pruning Score ($S^-$):** A scalar value per Gaussian representing its negative contribution to the training loss. A higher $S^-$ indicates the Gaussian contributes more to high-error (low-quality) regions, flagging it as an artifact or redundant point [S2].
- **Quantile Threshold ($Q_\tau^-(S^-)$):** The $\tau$-quantile of the distribution of $S^-$ values across all Gaussians [S2].

**Steps:**
1. **Synchronize with Training View Queue:**
   - Track the set of training views seen since the last pruning decision.
   - A pruning decision is triggered only after every view in the training set has been seen at least once since the last decision [S1, unofficial]. This ensures Gaussians in rarely observed regions are not pruned due to lack of opportunity [S1, unofficial].

2. **Calculate Pruning Score ($S^-$):**
   - For each Gaussian $i$, compute $S^-$ based on its impact on low-error pixel regions across multiple views [S2].
   - [G] Implementation note: In practice, this often involves aggregating the gradient magnitude or loss contribution of the Gaussian over the views it has been rendered in, weighted by the error in those regions. The source defines $S^-$ as a "multi-view and loss-driven strategy" where a larger $S^-$ indicates higher impact on low-error regions [S2].

3. **Compute Quantile Threshold:**
   - Calculate $Q_\tau^-(S^-)$, the quantile of the pruning score distribution [S2].
   - [G] The specific quantile $\tau$ is not explicitly defined in the provided notes, but it serves as the cutoff for the "moderate" aspect, allowing some low-opacity Gaussians to survive if their contribution score is low [S2].

4. **Generate Pruning Mask:**
   - Apply the following logical condition to each Gaussian $i$:
     $$ \mathcal{M}_{prune} = (o_i < o_{min}) \land \left( S^- > Q_\tau^-(S^-) \right) $$
     where:
     - $o_i$ is the opacity of Gaussian $i$.
     - $o_{min}$ is the opacity threshold [S2].
     - $S^-$ is the pruning score for Gaussian $i$ [S2].
     - $Q_\tau^-(S^-)$ is the quantile threshold [S2].
   - If the condition is true, the Gaussian is marked for pruning [S2].

5. **Postpone Pruning to Densification Event:**
   - Record the pruning mask $\mathcal{M}_{prune}$ when the decision is made [S1, unofficial].
   - Do not remove Gaussians immediately. Postpone the actual removal to the next densification event (splitting/cloning) [S1, unofficial]. This aligns pruning with the densification schedule, avoiding misalignment between the training queue and densification intervals [S1, unofficial].

6. **Execute Pruning:**
   - After densification finishes, remove all Gaussians marked in $\mathcal{M}_{prune}$ [S1, unofficial].
   - Contribution pruning continues to remove redundant Gaussians on its own in subsequent cycles [S1, unofficial].

7. **Post-Training Cleanup:**
   - After training completes, iterate once through all training views for a final contribution prune to ensure no low-contribution Gaussians remain from the final iterations [S1, unofficial].

**Parameters:**
- **$o_{min}$:** The opacity threshold below which Gaussians are considered for pruning. Typically set to 0.005, conservatively equal to the standard opacity prune threshold [S1, unofficial][S2].
- **$\tau$:** The quantile parameter for $Q_\tau^-(S^-)$. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Rarely Observed Regions:** Gaussians in areas rarely seen by the camera may have low $S^-$ simply because they haven't been rendered enough. The synchronization with the full training view queue prevents this [S1, unofficial].
- **Aggressive Pruning:** Using only $o_{min}$ leads to a performance slump. Moderate Pruning mitigates this by requiring $S^-$ to exceed the quantile threshold, preserving essential geometry [S2].
- **Misalignment:** Running pruning independently of densification can cause misalignment. Integrating the mask recording with the densification event avoids this [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Adds overhead to the training loop for calculating $S^-$ and tracking view visibility. The pruning itself is $O(N)$ where $N$ is the number of Gaussians, but is deferred to densification events, which are already $O(N)$ operations.
- **Memory Complexity:** Requires storing $S^-$ for each Gaussian and tracking view visibility, adding $O(N)$ memory overhead.
- **Performance:** Prevents the "performance slump" associated with standard opacity pruning by preserving geometrically significant but low-opacity Gaussians [S2]. Improves reconstruction fidelity in occluded regions [S1, unofficial].
- **Strengths:** Handles occlusion better than opacity-only methods; reduces redundant Gaussians without sacrificing quality; integrates seamlessly with existing densification pipelines.
- **Limitations:** Requires calculating loss-driven scores, which may be computationally more expensive than simple opacity checks; depends on the synchronization of the training view queue.

## Background
Moderate Pruning was introduced in the context of 3D Gaussian Splatting optimization to address the limitations of standard opacity-based pruning. It builds on [Opacity Pruning](14627-opacity-pruning.md) by adding a contribution-based criterion. The method is described in [S2] as a way to achieve "moderate pruning" by using a multi-view, loss-driven strategy. It is also referenced in [S1, unofficial] as "Contribution-Prune," noting its use in other works like RadSplat and ReLaGS, and its integration with occlusion-aware densification.

## Sources
[S1] Counting Contributions, Not Views: Occlusion-Aware Densification for 3D Gaussian Splatting, https://arxiv.org/abs/2610.27108, fetched 2026-10-26 (unofficial)
[S2] Monte Carlo Energy Aggregation for Mobile 3D Gaussian Splatting, https://doi.org/10.1007/978-3-032-37167-6_17, fetched 2026-10-29
