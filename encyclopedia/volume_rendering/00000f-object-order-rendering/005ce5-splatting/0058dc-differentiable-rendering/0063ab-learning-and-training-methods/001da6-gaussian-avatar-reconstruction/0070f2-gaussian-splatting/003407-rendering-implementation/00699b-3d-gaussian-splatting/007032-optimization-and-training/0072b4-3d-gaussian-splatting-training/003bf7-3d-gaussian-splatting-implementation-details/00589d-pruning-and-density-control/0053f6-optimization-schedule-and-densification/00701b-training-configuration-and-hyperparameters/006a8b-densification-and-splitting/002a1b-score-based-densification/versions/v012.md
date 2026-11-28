# Score-based Densification

Also called: Budget-constrained Optimization, Guided Densification

## Overview
Score-based Densification is a constructive training strategy for 3D Gaussian Splatting that controls model size by adding new primitives only to those Gaussians that maximize a specific quality score, ensuring the final model adheres to a user-defined budget without requiring pruning [S1]. Unlike standard 3D Gaussian Splatting which relies on heuristic thresholds and subsequent culling, this method uses a purely constructive schedule that steers densification toward Gaussians contributing most to reconstruction quality, avoiding optimization instability caused by aggressive removal [S1].

## How it works
**Inputs:** A set of 3D Gaussians $\mathcal{G}$, a set of training views, a target model size budget $B$, and a growth curve function $N(t)$ [S1].
**Outputs:** A refined set of Gaussians $\mathcal{G}'$ with size $|\mathcal{G}'| \approx B$ [S1].

**Data Structures:**
*   **Gaussian:** Inherits standard 3DGS fields: mean $\mu_k$, covariance $\Sigma_k$, opacity $\alpha_k$, spherical harmonics $\mathbf{c}_k$ [G].
*   **Score Buffer:** A scalar value $S_g$ stored per Gaussian, initialized to zero [S1].
*   **Saliency Map:** A 2D tensor per view representing pixel-wise importance [S1].

**Steps:**
1.  **Sample Views:** Select a batch of training views for the current iteration [S1].
2.  **Compute Saliency:** Render the current Gaussians and compute per-pixel saliency to identify important regions [S1].
3.  **Compute Per-Gaussian Score ($S_g$):** For each Gaussian $g_k$, aggregate metrics into a scoring function $F$ [S1]. The score combines:
    *   Gradient magnitude: Accumulated 2D positional gradients $\tau_k$ [G].
    *   Saliency: The sum of saliency values projected onto the Gaussian's footprint [S1].
    *   Primitive properties: Current scale and opacity [S1].
    $$ S_g(g_k) = F(\tau_k, \text{saliency}_k, \text{properties}_k) $$
    [G] The exact weighting of $F$ is implementation-dependent but must prioritize gradients in high-saliency regions.
4.  **Select Top Gaussians:** Rank all Gaussians by $S_g$ in descending order. Select the top Gaussians such that the total number of Gaussians after splitting matches the target count $N(t)$ defined by the growth curve [S1].
5.  **Densify:** For each selected Gaussian $g_k$:
    *   **Clone:** Create a copy of $g_k$ with identical parameters $\mu, \Sigma, \alpha, \mathbf{c}$ [G].
    *   **Split:** Replace $g_k$ with children sampled from $\mathcal{N}(\mu_k, \Sigma_k)$ [G].
    *   [G] The number of children per split is determined by the remaining budget and the current count. Typically, one clone and one split (or similar) are performed to double the local density.
6.  **Update Growth Curve:** Increment the iteration counter. The target size $N(t)$ follows a curve mimicking standard 3DGS behavior but fitted to reach exactly $B$ at the end of training [S1].

**Parameters:**
*   **Budget $B$:** The final number of Gaussians [S1].
*   **Growth Curve:** A function $N(t)$ mapping iteration $t$ to target count [S1].

**Edge Cases:**
*   **Budget Exhaustion:** If the current count exceeds $B$ before the end of training, no further densification occurs [G].
*   **Degenerate Gaussians:** Gaussians with near-zero opacity or extremely small scales should be excluded from scoring to prevent noise amplification [G].

## Subtopics

### [Pixel-GS](9189-pixel-gs.md)
Pixel-GS is a densification strategy for 3D Gaussian Splatting that modifies the gradient-based splitting signal to account for the spatial footprint of each Gaussian primitive. It addresses the baseline's tendency to under-densify regions by scaling the positional gradient by the number of pixels a Gaussian influences and applying a depth-dependent scaling factor to suppress floaters. It yields state-of-the-art rendering quality compared to standard 3DGS, though it typically results in a higher number of Gaussian primitives. It is chosen when visual fidelity is prioritized over strict budget constraints.

### Perceptual-GS
Perceptual-GS replaces the simple gradient threshold densification heuristic with a scene-adaptive perceptual signal to improve detail in visually salient areas [S2, unofficial]. It computes a local perceptual error map using a lightweight vision-transformer head and densifies only in regions where Structural Similarity Index Measure (SSIM) falls below a dynamic threshold [S2, unofficial]. It is chosen when perceptual quality metrics are more critical than geometric gradient accuracy.

### Late Densification
Late Densification is a temporal strategy that enables densification back between iterations 20K to 20.5K for every 100 iterations to help the model recover loss from pruning of false-positive Gaussians [S3]. It is applied as a supplement to any densification strategy to refine details after the main structure has converged.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method addresses training speed obstacles by proposing faster, numerically equivalent solutions for gradient computation and attribute updates [S1].
*   **Memory Complexity:** Not given in the sources. The method ensures predictable memory usage bounded by $B$ [S1].
*   **Performance:** Eliminates the need for pruning/culling passes, reducing training time variability and avoiding optimization peaks [S1]. Ensures predictable memory usage bounded by $B$ [S1].
*   **Strengths:** Controlled model size, no post-training culling required, stable optimization [S1].
*   **Limitations:** Not given in the sources.

## Background
Score-based Densification was introduced to address the unpredictable storage demands and fluctuating training times of standard 3D Gaussian Splatting [S1]. It was proposed in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" (2024) as a guided, purely constructive alternative to heuristic-based densification [S1]. The method builds on the observation that redundancy in 3DGS can be managed by steering additions toward high-quality-contributing primitives rather than removing low-quality ones [S1].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
