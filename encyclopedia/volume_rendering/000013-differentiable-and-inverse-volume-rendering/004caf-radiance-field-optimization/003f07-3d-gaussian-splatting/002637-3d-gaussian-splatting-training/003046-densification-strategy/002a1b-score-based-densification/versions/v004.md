# Score-based Densification

Also called: Budget-constrained Optimization, Guided Densification

## Overview
Score-based Densification is a constructive densification strategy for 3D Gaussian Splatting that steers the addition of new primitives toward those that maximize reconstruction quality, ensuring the model size increases in a controlled manner toward an exact user-defined budget [S1]. Unlike standard adaptive density control which relies on gradient thresholds and subsequent pruning, this method uses a purely constructive schedule, avoiding the optimization instability and hardware constraint violations caused by substantial pruning or culling during training [S1].

## How it works
The method replaces the heuristic threshold-based splitting of standard 3DGS with a guided, score-driven selection process. It operates by sampling training views, computing per-pixel saliency, and aggregating these into a per-Gaussian score to determine which primitives to densify [S1].

**Inputs:**
*   Current set of Gaussians $\mathcal{G}$.
*   Training views $V_{train}$.
*   Target final model size (budget) $N_{budget}$.
*   Growth curve parameters defining the densification schedule.

**Outputs:**
*   Updated set of Gaussians $\mathcal{G}'$ with increased count.
*   New Gaussians added in the vicinity of high-scoring primitives.

**Data Structures:**
*   **Gaussian Buffer:** Stores mean $\mu$, covariance $\Sigma$, opacity $\alpha$, spherical harmonics coefficients, and scale $s$.
*   **Saliency Map:** Per-pixel importance metric derived from loss gradients.
*   **Score List:** Sorted list of Gaussians ranked by their computed score $S_g$.

**Step-by-Step Method:**

1.  **Sample Views:** Select a subset of training views $V_{sample} \subset V_{train}$ for the current densification step.
2.  **Compute Per-Pixel Saliency:** Render the current Gaussians and compute the reconstruction loss $L$ against ground truth images. Calculate the gradient of the loss with respect to the image plane coordinates to identify regions of high error or importance. This yields a per-pixel saliency map [S1].
3.  **Compute Per-Gaussian Score ($S_g$):** For each Gaussian $i$, compute a score $S_g^{(i)}$ using a scoring function $F$ that combines:
    *   **Gradient Magnitude:** The magnitude of the gradient of the loss with respect to the Gaussian's parameters (specifically the 2D mean position or relevant attribute gradients) [S1].
    *   **Saliency:** The aggregated saliency value of the pixels covered by the Gaussian's projection [S1].
    *   **Primitive Properties:** Features such as scale, opacity, or current contribution to the image [S1].
    
    The exact formula for $F$ is not explicitly defined in the sources, but it is described as a combination of these loss-relevant components collected across sampled views [S1].
4.  **Select Top-Scoring Gaussians:** Rank all Gaussians by $S_g$ in descending order. Select the top $K$ Gaussians where $K$ is determined by the current step in the growth curve and the target budget [S1].
5.  **Densify (Constructive Step):** For each selected Gaussian, create new Gaussians in its vicinity. The sources describe this as "densification... in the vicinity of the top-scoring Gaussians" [S1]. Standard 3DGS splitting logic (perturbing mean along principal axes, reducing scale) is applied to these selected primitives [S1].
6.  **Follow Growth Curve:** The number of Gaussians added at each step is governed by a growth curve that mimics the behavior of standard 3DGS but is fitted to yield the exact desired model size $N_{budget}$ after training completes [S1]. This ensures a predictable, monotonic increase in primitive count without overshooting the budget.
7.  **No Pruning:** The process is purely constructive. No Gaussians are pruned or culled during this phase [S1].

**Parameters:**
*   **Budget ($N_{budget}$):** The target final number of Gaussians.
*   **Growth Curve:** A function mapping training step to cumulative Gaussian count.
*   **Sampling Rate:** Number of views sampled per densification step.

**Edge Cases and Pitfalls:**
*   **Gradient Computation:** The method requires efficient gradient computation. The source notes that they derive "faster, numerically equivalent solutions for gradient computation and attribute updates" to handle the overhead of this scoring process [S1].
*   **Numerical Equivalence:** Approximations are used where suitable to reduce training time, but must preserve the correctness of the score calculation [S1].

## Subtopics

### [Pixel-GS](9189-pixel-gs.md)
Pixel-GS is a variant that prioritizes splitting Gaussians based on their spatial impact on the rendered image. It scales the densification gradient by the number of pixels a Gaussian influences, ensuring that primitives causing errors across many pixels are split first. Unlike standard 3DGS which uses average positional gradients, Pixel-GS accounts for the projected area and depth, yielding state-of-the-art rendering quality at the cost of a higher total number of Gaussian primitives.

### Perceptual-GS
Perceptual-GS replaces the simple gradient threshold densification heuristic with a scene-adaptive perceptual signal to improve detail in visually salient areas [S2, unofficial]. It computes a local perceptual error map using a lightweight vision-transformer head and densifies Gaussians only in regions where the Structural Similarity Index Measure (SSIM) falls below a dynamic threshold [S2, unofficial].

### Late Densification
Late Densification is a scheduling variant that enables densification in a narrow window late in training (e.g., between iterations 20K to 20.5K) for every 100 iterations [S3]. This is intended to help the model recover loss from the pruning of false-positive Gaussians that may have occurred earlier in the training process [S3].

## Operating characteristics
*   **Time Complexity:** The method introduces overhead due to the computation of per-pixel saliency and the aggregation of scores across views. However, the source notes that they derive "faster, numerically equivalent solutions" for gradient computation to mitigate this [S1].
*   **Memory Complexity:** Memory usage is predictable and bounded by the user-defined budget $N_{budget}$, unlike standard 3DGS which can produce unpredictable peak memory usage due to uncontrolled splitting and pruning [S1].
*   **Performance:** The method ensures high-quality reconstruction at a fixed budget. It avoids the "unnecessary peaks in optimization" associated with pruning-heavy methods [S1].
*   **Strengths:** Controlled model size, no need for pruning/culling, predictable training duration, better suitability for downstream tasks requiring fixed-size inputs [S1].
*   **Limitations:** Requires careful tuning of the growth curve and scoring function. May be slower per iteration than standard 3DGS due to saliency computation [S1].

## Background
Score-based Densification was introduced in the paper "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" [S1]. It addresses the challenges of unpredictable storage demands and fluctuating training times in standard 3D Gaussian Splatting, which make it difficult to use in budget-constrained scenarios or downstream tasks [S1]. The method was developed to provide a systematic redesign of the densification process, moving away from heuristic thresholds toward a guided, score-based approach [S1].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
