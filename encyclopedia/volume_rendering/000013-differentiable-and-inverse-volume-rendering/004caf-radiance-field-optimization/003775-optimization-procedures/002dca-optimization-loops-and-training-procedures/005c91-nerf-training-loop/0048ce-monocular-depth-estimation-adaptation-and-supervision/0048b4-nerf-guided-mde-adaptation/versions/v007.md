# NeRF-guided MDE Adaptation

Also called: MDE Adaptation via NeRF, Scene-specific MDE fine-tuning

## Overview
NeRF-guided MDE Adaptation is a method that refines a pre-trained Monocular Depth Estimation (MDE) network by using a NeRF model as a geometric prior to resolve ambiguities inherent in relative depth predictions, such as convexity/concavity confusion and flat surface orientation errors [S1, unofficial]. It distinguishes itself from sibling methods like [NerfingMVS](9518-nerfingmvs.md) by explicitly adapting the MDE weights to the specific scene's absolute geometry rather than just using depth as a static supervision signal for the NeRF.

## How it works
**Inputs:**
*   Input RGB images $I_i$ for seen views $S$ and potentially unseen views $U$.
*   Pre-trained MDE network $G_\phi$ (e.g., DPThybrid) [S1, unofficial].
*   NeRF model $F_\Theta$ (e.g., K-planes) [S1, unofficial].
*   Camera poses and intrinsics.

**Outputs:**
*   Adapted MDE weights $\phi$.
*   Optimized NeRF weights $\Theta$.
*   Depth-consistent novel view images.

**Data Structures:**
*   **MDE Output:** Relative depth map $D^*$ and initial relative depth map $D^*_{init}$ (extracted before adaptation starts).
*   **NeRF Output:** Rendered absolute depth map $\bar{D}$ and color $C$.
*   **Confidence Mask:** $M(p)$, a scalar value per pixel indicating reliability [S1, unofficial].

**Parameters:**
*   Learning rate for NeRF: $1 \cdot 10^{-2}$ [S1, unofficial].
*   Learning rate for MDE: $1 \cdot 10^{-5}$ [S1, unofficial].
*   Optimizer: Adam with cosine warmup scheduling [S1, unofficial].
*   Scale $w_i$ and shift $q_i$: Per-view affine parameters.

**Steps:**
1.  **Initialize:** Extract initial monocular depth $D^*_{init}$ from $G_\phi$ using pre-trained weights. Initialize NeRF $F_\Theta$.
2.  **Render NeRF Depth:** For each seen view $I_i$, render the absolute depth map $\bar{D}_i$ by accumulating sample depths weighted by transmittance and density along rays [S1, unofficial].
3.  **Extract MDE Depth:** Process $I_i$ through $G_\phi$ to get relative depth $D^*_i$.
4.  **Compute Alignment:** Calculate scale $w_i$ and shift $q_i$ via least-squares optimization to align $D^*_i$ with $\bar{D}_i$:
    $$ \min_{w_i, q_i} \sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \|^2 $$
    where $\text{sg}(\cdot)$ is the stop-gradient operator [S1, unofficial].
5.  **Compute Losses:** Calculate the total loss (see Formulas below).
6.  **Update:** Perform gradient descent to update both $\Theta$ and $\phi$ simultaneously [S1, unofficial].

**Formulas:**
The core MDE adaptation loss $L_{MDE}$ combines an absolute alignment term (without scale/shift) and a patch-wise aligned term [S1, unofficial]:
$$ L_{MDE} = \sum_{I_i \in S} \sum_{p \in P} \left( \| \text{sg}(\bar{D}_i(p)) - D^*_i(p) \| + \| (w_i \cdot \text{sg}(\bar{D}_i(p)) + q_i) - D^*_i(p) \| \right) $$
Note: The source text [S1, unofficial] uses $D^*_i(p)$ in the first term of Eq. 6, but later Eq. 12 uses $\bar{D}^*_i(p)$ (likely a typo for $D^*_i$ or referring to the aligned version). Based on the description "adapt the MDE with absolute depth prior" and the structure of sibling methods, the first term aligns the raw MDE output to the NeRF depth without affine adjustment, while the second aligns the affine-adjusted MDE output. We follow Eq. 6 strictly as written in the source for the base method.

A regularization term $L_{reg}$ preserves the local smoothness of the MDE adaptation by comparing the adapted depth to the initial pre-trained depth [S1, unofficial]:
$$ L_{reg} = \sum_{I_i \in S} \sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_{init}(p)) + q_i) - D^*_i(p) \| $$

**Pitfalls:**
*   MDE lacks awareness of scene-specific absolute depth and multi-view consistency, leading to ambiguities in surface convexity/concavity and flat surface orientation [S1, unofficial].
*   Errors in MDE or NeRF can propagate and intensify during distillation if not controlled [S1, unofficial].

## Subtopics

### Overall Training with Confidence
This variant incorporates confidence masks $M(p)$ to weight the depth consistency losses, preventing error propagation from inaccurate depths [S1, unofficial]. It trains NeRF and MDE simultaneously, where MDE provides geometric priors to NeRF and NeRF provides absolute geometry to MDE [S1, unofficial]. The losses are weighted by confidence maps $M_i(p)$ for seen views and $M_l(p)$ for unseen views [S1, unofficial].

The loss for seen views $L_{seen}$ is:
$$ L_{seen} = \sum_{I_i \in S} \sum_{p \in P} M_i(p) \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \| $$
The loss for unseen views $L_{unseen}$ is:
$$ L_{unseen} = \sum_{I_l \in U} \sum_{p \in P} M_l(p) \| (w_l \cdot \text{sg}(\bar{D}^*_l(p)) + q_l) - \bar{D}_l(p) \| $$
The MDE adaptation loss becomes:
$$ L_{MDE} = \sum_{I_i \in S} \sum_{p \in P} M_i(p) \left( \| \text{sg}(\bar{D}_i(p)) - \bar{D}^*_i(p) \| + \| (w_i \cdot \text{sg}(\bar{D}_i(p)) + q_i) - \bar{D}^*_i(p) \| \right) $$
Note: The source [S1, unofficial] uses $\bar{D}^*_i(p)$ in Eq. 12, which likely denotes the MDE output in the context of the confidence-weighted loss, consistent with the pattern of sibling methods.

## Operating characteristics
*   **Time Complexity:** Increases training time due to the simultaneous optimization of both NeRF and MDE networks. The least-squares alignment step adds negligible overhead per iteration.
*   **Memory Complexity:** Requires storing gradients for both NeRF and MDE, increasing VRAM usage compared to fixed-MDE approaches.
*   **Performance:** Improves NeRF reconstruction quality (PSNR, SSIM) on sparse input sets (e.g., 9-10 views on ScanNet) by providing better geometric priors [S1, unofficial].
*   **Strengths:** Resolves MDE ambiguities (convexity/concavity) and adapts to scene-specific geometry.
*   **Limitations:** Requires careful tuning of learning rates (NeRF LR >> MDE LR) to prevent instability. Confidence modeling adds complexity.

## Background
Originated in the DäRF paper (2023) [S1, unofficial]. It addresses the limitation of standard NeRF training with monocular depth where the MDE provides only relative depth, leading to geometric inconsistencies. By adapting the MDE to the NeRF's absolute geometry, it creates a mutually beneficial training loop.

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
