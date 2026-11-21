# Score-based Densification

Also called: Budget-constrained Optimization, Guided Densification

## Overview
Score-based Densification is a constructive training strategy for 3D Gaussian Splatting that manages primitive count by adding new Gaussians only where they improve reconstruction quality, adhering to a strict, user-defined budget [S1]. Unlike standard 3DGS which relies on reactive pruning and culling to manage size, this method uses a scoring function to guide a purely additive schedule, ensuring the final model size is predictable and hardware-constrained without intermediate peaks in optimization [S1]. It builds on [Densification Strategy](6234-densification-strategy.md) by replacing gradient-threshold triggers with a multi-component score.

## How it works
**Inputs:** Training views, camera poses, initial Gaussian set, target budget $B$ (final number of Gaussians), and a growth curve function $N(t)$.
**Outputs:** A set of $N_{final} \le B$ Gaussians optimized for reconstruction.

**Data Structures:**
*   **Gaussian $g_k$**: Contains position $\mu_k$, covariance $\Sigma_k$ (or scale $s_k$, rotation $q_k$), opacity $\alpha_k$, spherical harmonics coefficients $c_k$.
*   **Score Buffer**: Stores per-Gaussian score $S_g$ and per-pixel saliency maps.

**Scoring Function:**
The per-Gaussian score $S_g$ is computed by combining loss-relevant components collected across sampled training views [S1]. While the exact formula for $F$ is not fully expanded in the provided notes, it combines:
1.  **Gradient:** Magnitude of the positional gradient $\frac{\partial L_v}{\partial \mu_v^k}$ [S1].
2.  **Saliency:** Per-pixel saliency derived from training views [S1].
3.  **Primitive Properties:** Geometric or opacity metrics of $g_k$ [S1].

**Steps:**
1.  **Sample Views:** Select a batch of training views for the current optimization step [S1].
2.  **Compute Saliency:** Render the current scene and compute per-pixel saliency to identify regions contributing most to the loss or requiring detail [S1].
3.  **Compute Scores:** For each Gaussian $g_k$, calculate its score $S_g$ using the function $F$ which aggregates gradient, saliency, and primitive properties [S1].
4.  **Select Candidates:** Identify the top-scoring Gaussians based on $S_g$ [S1].
5.  **Densify:** Add new Gaussians in the vicinity of the top-scoring candidates. The number of added Gaussians is determined by the growth curve $N(t)$, which mimics the behavior of standard 3DGS but is fitted to reach the exact budget $B$ at the end of training [S1].
6.  **Optimize:** Update Gaussian parameters using backpropagation [S1].
7.  **Repeat:** Continue until the budget $B$ is reached or training completes.

**Parameters:**
*   **Budget $B$:** The target final number of Gaussians. Not given in the sources.
*   **Growth Curve:** A function $N(t)$ defining the number of Gaussians at step $t$. Must be fitted to yield $B$ at $t_{end}$ [S1].

**Edge Cases:**
*   **Budget Exhaustion:** If the budget is reached before training converges, densification stops, but optimization continues for existing Gaussians [S1].
*   **Redundancy:** The method avoids redundant primitives by construction, reducing the need for pruning [S1].

## Subtopics

### Perceptual-GS
Perceptual-GS replaces the simple gradient threshold with a scene-adaptive perceptual signal to improve detail in visually salient areas [S2, unofficial]. It computes a local perceptual error map using a lightweight vision-transformer head and densifies Gaussians only in regions where the Structural Similarity Index Measure (SSIM) falls below a dynamic threshold [S2, unofficial].

### Late Densification
Late Densification is a temporal adjustment strategy where densification is re-enabled between iterations 20K and 20.5K for every 100 iterations [S3]. This helps the model recover loss from the pruning of false-positive Gaussians that occurred earlier in training [S3].

## Operating characteristics
*   **Complexity:** Time complexity is dominated by the scoring computation and backpropagation, similar to standard 3DGS but with additional saliency calculation [S1]. Memory complexity is strictly bounded by the budget $B$, avoiding the large intermediate peaks of pruning-based methods [S1].
*   **Performance:** Provides predictable model sizes and training durations [S1]. Avoids hardware constraint violations caused by transient large model sizes [S1].
*   **Strengths:** Purely constructive; no pruning required during training; exact budget control; reduced redundancy [S1].
*   **Limitations:** Requires careful fitting of the growth curve; may be slower to converge if the score function is expensive [S1].
*   **When to use:** When hardware memory is limited, or when a fixed-size model is required for downstream tasks [S1].

## Background
Score-based Densification was introduced in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" [S1]. It addresses the challenges of unpredictable storage demands and fluctuating training times in standard 3DGS [S1]. The method was developed to enable high-quality radiance field synthesis under strict resource constraints, offering a guided, constructive alternative to the reactive pruning and culling of earlier approaches [S1].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
