# Adaptive Density Control (Split & Clone)

Also called: Clone, Split, ADC

## Overview
Adaptive Density Control (ADC) is a densification strategy used during the optimization of 3D Gaussian Splatting to adjust the local density of Gaussians to match scene complexity [S1, S2, unofficial]. It solves the problem of insufficient representation in complex regions (under-densification) by either duplicating or subdividing Gaussians based on their coverage relative to the target reconstruction region [S1, unofficial]. It is distinguished from its siblings by its dual-mode approach: it explicitly clones Gaussians that are too small to cover a region and splits those that are too large, whereas variants like [Density-preserving Gaussian splitting](9290-density-preserving-gaussian-splitting.md) or [3DGS Adaptive Density Control](7847-3dgs-adaptive-density-control.md) may drop the clone operation or use different sampling strategies [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$, each defined by mean $\mu_i$, covariance $\Sigma_i$ (or scale $\sigma_i$ and rotation $R_i$), opacity $\alpha_i$, and spherical harmonic coefficients $c_i$. A binary mask indicating which Gaussians are marked for densification.
*   **Output:** An updated set of Gaussians where marked Gaussians are replaced by either two children (Split) or two identical copies (Clone).

**Data Structures:**
Each Gaussian $G$ stores:
*   $\mu \in \mathbb{R}^3$: Mean position.
*   $\sigma \in \mathbb{R}^3$: Scale vector (diagonal of covariance).
*   $q \in \mathbb{R}^4$: Rotation quaternion.
*   $\alpha \in \mathbb{R}$: Opacity.
*   $c \in \mathbb{R}^k$: SH coefficients.

**Steps:**
1.  **Evaluate Coverage:** For each Gaussian marked for densification, compare its spatial extent (coverage) against the target reconstruction region or scene extent [S1, S2, unofficial].
2.  **Classify:**
    *   If the Gaussian is **too small** (Under-Reconstruction), mark it for **Clone** [S1, unofficial].
    *   If the Gaussian is **too large** (Over-Reconstruction), mark it for **Split** [S1, unofficial].
3.  **Execute Clone:**
    *   Create two new Gaussians $G_{c1}, G_{c2}$ identical to the parent $G_p$ in all attributes ($\mu, \sigma, q, \alpha, c$) [S1, unofficial].
    *   [G] In some implementations, slight perturbations may be added to means to prevent exact overlap, but the core definition is duplication [S2, unofficial].
4.  **Execute Split:**
    *   Follow the **Splitting Mechanics and Density Preservation** [Parent] protocol:
        *   Identify the principal axis with the largest scale $\sigma_{max}$.
        *   Shift the means of the two children along this axis by $\Delta = \alpha_{shift} \cdot \sigma_{max}$ (where $\alpha_{shift}$ is a constant, typically 0.5 or similar, though exact value not given in sources [Parent]).
        *   Copy rotation and SH coefficients from parent to children [Parent].
        *   Adapt scales and opacities to preserve density [Parent].
    *   [S2, unofficial] Note: Some variants sample child positions from the parent's distribution rather than deterministic shifts, but the standard ADC definition relies on the split mechanics described in the parent article.

**Parameters:**
*   **Thresholds:** The criteria for "too small" vs "too large" are relative to the scene extent or local detail size [S1, S2, unofficial]. Exact numerical thresholds are not given in the sources.
*   **Opacity Reset:** To clear floaters, opacities may be reset to near zero (e.g., 0.01) every 3000 iterations [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Opacity Bias:** Cloning introduces an opacity bias because two identical Gaussians with opacity $\alpha$ effectively contribute more density than one, potentially leading to over-reconstruction [S2, unofficial].
*   **Floater Clusters:** Excessive cloning on scenes where the scene extent is not well-matched to the size of detail can cause large clusters of floating artifacts [S2, unofficial].
*   **Thin Structures:** Removing cloning breaks thin structures (e.g., bicycle spokes), causing them to appear fragmented [S1, unofficial].

## Subtopics

### Split-Only Densification
A variant that drops the clone operation entirely, using only splitting [S2, unofficial]. If a Gaussian is marked for densification and is large, it is split into two downscaled children sampled from the parent's distribution [S2, unofficial]. If it is small, it applies a geometrically more meaningful split procedure (e.g., from Deng et al.) rather than cloning [S2, unofficial]. This approach avoids the opacity bias and floater clusters associated with cloning but requires careful handling of small Gaussians [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of marked Gaussians. Cloning is $O(1)$ per Gaussian; Splitting is $O(1)$ but involves more arithmetic for scale/rotation adaptation.
*   **Memory Complexity:** Increases the total number of Gaussians, doubling the count for cloned/split items.
*   **Performance:** On the Average-30K dataset, the full method achieves a PSNR of 26.05 [S1, unofficial]. Removing Split drops PSNR to 23.90, while removing Clone drops it to 25.91 [S1, unofficial]. This indicates Split has a larger impact on overall image quality, particularly background coherence, while Clone is critical for thin structures [S1, unofficial].
*   **Strengths:** Effectively handles both under-densified (small detail) and over-densified (blurry) regions [S1, unofficial].
*   **Limitations:** Cloning can introduce opacity bias and floaters if scene extent estimation is poor [S2, unofficial].

## Background
The method was introduced in the original 3D Gaussian Splatting paper (Kerbl et al., 2023) as "Adaptive Density Control" [S1, unofficial]. It is a core component of the training optimization loop, allowing the static set of initial Gaussians to adapt to the geometric and textural complexity of the scene during differentiable rendering optimization [S1, unofficial].

## Sources
[S1] GaussianSplatting論文のAblationについての解説・考察, https://qiita.com/kotauchisunsun/items/f1e89f28aa770b8d3b19, fetched 2026-10-29 (unofficial)
[S2] Counting Contributions, Not Views: Occlusion-Aware Densification for 3D Gaussian Splatting, https://arxiv.org/abs/2610.27108, fetched 2026-10-26 (unofficial)
