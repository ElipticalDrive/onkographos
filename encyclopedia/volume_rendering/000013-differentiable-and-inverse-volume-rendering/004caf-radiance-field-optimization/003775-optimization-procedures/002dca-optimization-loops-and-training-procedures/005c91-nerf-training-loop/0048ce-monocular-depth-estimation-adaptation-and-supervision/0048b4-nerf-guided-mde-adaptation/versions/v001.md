# NeRF-guided MDE Adaptation

Also called: MDE Adaptation via NeRF, Scene-specific MDE fine-tuning

## Overview
NeRF-guided MDE Adaptation is a method to adapt a pre-trained Monocular Depth Estimation (MDE) network to the specific absolute geometry of a scene using a Neural Radiance Field (NeRF) as a geometric prior [S1, unofficial]. It solves the ambiguity inherent in monocular depth estimation, such as the inability to distinguish surface convexity/concavity or determine the orientation of flat surfaces, by leveraging the NeRF's knowledge of canonical space and absolute geometry [S1, unofficial]. Unlike sibling methods that primarily use depth to regularize NeRF training, this method explicitly fine-tunes the MDE weights to align with the rendered NeRF depth, creating a bidirectional knowledge transfer where the MDE provides geometric priors to the NeRF and the NeRF provides absolute scale/orientation to the MDE [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of seen views $S$, a pre-trained MDE model, and a NeRF model optimized for the scene.
*   **Outputs:** Adapted MDE weights and refined NeRF parameters.

**Data Structures:**
*   $D_i^*$: Monocular depth map for view $I_i$ predicted by the current MDE.
*   $\bar{D}_i$: NeRF-rendered depth map for view $I_i$.
*   $D_{i,init}^*$: Monocular depth map for view $I_i$ extracted from the MDE using its initial pre-trained weights (used for regularization).
*   $w_i, q_i$: Scale and shift parameters computed via least squares.
*   $P$: A set of patches or pixels used for loss computation.

**Steps:**
1.  **Render NeRF Depth:** For each seen view $I_i \in S$, render the depth map $\bar{D}_i$ from the NeRF.
2.  **Extract MDE Depth:** Extract the initial monocular depth $D_{i,init}^*$ from the MDE using pre-trained weights.
3.  **Compute Alignment Parameters:** Compute scale $w_i$ and shift $q_i$ via least squares fitting between the current MDE prediction $D_i^*$ and the NeRF depth $\bar{D}_i$ [S1, unofficial].
4.  **Compute Losses:** Calculate the MDE adaptation loss ($L_{MDE}$) and regularization loss ($L_{reg}$) as defined below.
5.  **Update:** Backpropagate to update both the NeRF and MDE parameters simultaneously [S1, unofficial].

**Formulas:**
The MDE adaptation loss $L_{MDE}$ combines absolute alignment and patch-wise alignment, using a stop-gradient ($sg(\cdot)$) on the NeRF depth to prevent gradient flow into the NeRF during MDE updates [S1, unofficial]:

$$
L_{MDE} = \sum_{I_i \in S} \sum_{p \in P} \left( \| sg(\bar{D}_i(p)) - D_i^*(p) \| + \| (w_i \cdot sg(\bar{D}_i(p)) + q_i) - D_i^*(p) \| \right)
$$

The regularization term $L_{reg}$ preserves the local smoothness of the MDE adaptation by comparing the scaled initial depth to the current depth [S1, unofficial]:

$$
L_{reg} = \sum_{I_i \in S} \sum_{p \in P} \| (w_i \cdot sg(D_{i,init}^*(p)) + q_i) - D_i^*(p) \|
$$

**Parameters:**
*   **Learning Rate:** $1 \cdot 10^{-2}$ for NeRF, $1 \cdot 10^{-5}$ for MDE [S1, unofficial].
*   **Optimizer:** Adam [S1, unofficial].
*   **Scheduling:** Cosine warmup [S1, unofficial].

**Pitfalls:**
*   **Ambiguity:** MDE lacks awareness of scene-specific absolute depth and multi-view consistency, leading to errors in surface orientation [S1, unofficial].
*   **Error Propagation:** Inaccurate depths can propagate and intensify during distillation if not weighted by confidence [S1, unofficial].

## Subtopics

### Overall Training with Confidence
This variant incorporates confidence masks ($M_i(p)$ or $M_l(p)$) to weight the depth consistency losses, preventing error propagation from inaccurate depths [S1, unofficial]. It defines losses for seen views ($L_{seen}$) and unseen views ($L_{unseen}$), as well as the adapted MDE loss ($L_{MDE}$), all weighted by confidence [S1, unofficial]. The losses are defined as:

$$
L_{seen} = \sum_{I_i \in S} \sum_{p \in P} M_i(p) \| (w_i \cdot sg(D_i^*(p)) + q_i) - \bar{D}_i(p) \|
$$

$$
L_{unseen} = \sum_{I_l \in U} \sum_{p \in P} M_l(p) \| (w_l \cdot sg(\bar{D}_l^*(p)) + q_l) - \bar{D}_l(p) \|
$$

$$
L_{MDE} = \sum_{I_i \in S} \sum_{p \in P} M_i(p) \left( \| sg(\bar{D}_i(p)) - \bar{D}_i^*(p) \| + \| (w_i \cdot sg(\bar{D}_i(p)) + q_i) - \bar{D}_i^*(p) \| \right)
$$

Here, $U$ is the set of unseen views, and $\bar{D}_l^*$ is the MDE prediction for the unseen view [S1, unofficial]. The framework trains NeRF and MDE simultaneously, using MDE for geometric priors and NeRF for absolute geometry [S1, unofficial].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** On ScanNet (9-10 views), the method (DaRF) achieves a PSNR of 16.38, SSIM of 0.624, and LPIPS of 0.493 [S1, unofficial]. On Tanks and Temples (10 views), it achieves PSNR 21.54, SSIM 0.732, and LPIPS 0.168 [S1, unofficial].
*   **Strengths:** Resolves convexity/concavity ambiguities; provides absolute scale to MDE; improves NeRF reconstruction from sparse inputs [S1, unofficial].
*   **Limitations:** Relies on the quality of the initial NeRF and MDE; prone to error propagation without confidence modeling [S1, unofficial].
*   **When to use:** Sparse view reconstruction where monocular depth priors are beneficial but ambiguous [S1, unofficial].

## Background
The method was proposed in the paper "DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation" [S1, unofficial]. It addresses the ill-posed nature of monocular depth estimation by integrating it with NeRF optimization [S1, unofficial]. The approach builds on confidence modeling techniques from self-training approaches [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
