# NeRF-guided MDE Adaptation

Also called: MDE Adaptation via NeRF, Scene-specific MDE fine-tuning

## Overview
NeRF-guided MDE Adaptation is a method that aligns a pre-trained Monocular Depth Estimation (MDE) network with the absolute geometry of a specific scene by using a NeRF model as a geometric prior [S1, unofficial]. It solves the ambiguity inherent in monocular depth estimation, such as the inability to distinguish surface convexity/concavity or determine the orientation of flat surfaces, by transferring the NeRF's scene-specific absolute depth and multi-view consistency knowledge to the MDE [S1, unofficial]. Unlike sibling methods that only use MDE to regularize NeRF training, this method explicitly adapts the MDE parameters to the target scene, creating a bidirectional knowledge transfer where the MDE provides geometric priors to the NeRF and the NeRF provides absolute scale to the MDE [S1, unofficial].

## How it works
The method operates by iteratively refining the MDE network weights using depth consistency losses derived from the NeRF, while simultaneously training the NeRF.

**Inputs and Outputs:**
*   **Inputs:** A set of seen views $S$ with known camera poses and intrinsics; a pre-trained MDE network $G_\phi$ with initial weights; a NeRF model.
*   **Outputs:** Adapted MDE network weights $\phi$; optimized NeRF parameters.

**Data Structures:**
*   **NeRF Depth Map ($\bar{D}_i$):** Rendered depth for view $i$, computed as the expected distance: $\bar{D}_i(r) = \sum_j w_j y_j$, where $w_j$ is the transmittance-weighted contribution of sample $j$ at distance $y_j$ [S1, unofficial].
*   **MDE Depth Map ($D^*_i$):** Pseudo-depth map predicted by $G_\phi$ from image $I_i$.
*   **Initial MDE Depth Map ($D^*_{i,init}$):** Depth map from $G_\phi$ using initial pre-trained weights, used for regularization [S1, unofficial].
*   **Confidence Mask ($M(p)$):** A scalar value per pixel $p$ indicating reliability, derived from reprojection consistency or other confidence modeling techniques [S1, unofficial].

**Parameters:**
*   **Scale ($w_i$) and Shift ($q_i$):** Linear alignment parameters computed via least squares to map $D^*_i$ to $\bar{D}_i$.
*   **Learning Rates:** Typical values from implementation: $1 \cdot 10^{-2}$ for NeRF, $1 \cdot 10^{-5}$ for MDE [S1, unofficial].
*   **Optimizer:** Adam [S1, unofficial].

**Numbered Steps:**
1.  **Render NeRF Depth:** For each seen view $I_i \in S$, render the depth map $\bar{D}_i$ using the current NeRF model.
2.  **Extract MDE Depth:** Pass $I_i$ through the MDE network $G_\phi$ to obtain $D^*_i$.
3.  **Compute Alignment:** Calculate scale $w_i$ and shift $q_i$ by solving the least squares problem:
    $$ \min_{w_i, q_i} \sum_{p \in P} \| (w_i \cdot D^*_i(p) + q_i) - \bar{D}_i(p) \|^2 $$
    where $P$ is the set of pixels in the view [S1, unofficial].
4.  **Compute Losses:** Calculate the adaptation loss $L_{MDE}$ and regularization loss $L_{reg}$ (see formulas below). If confidence modeling is used, weight these losses by $M_i(p)$.
5.  **Update MDE:** Backpropagate the MDE-specific losses through $G_\phi$ to update weights $\phi$. Note that the stop-gradient operator $\text{sg}(\cdot)$ is applied to the NeRF depth $\bar{D}_i$ in the loss terms to prevent gradient flow into the NeRF during MDE updates [S1, unofficial].
6.  **Update NeRF:** Backpropagate the NeRF reconstruction losses (which may also be weighted by confidence) to update NeRF parameters.
7.  **Repeat:** Iterate steps 1-6 for the training duration.

**Formulas:**
The core adaptation loss $L_{MDE}$ combines an absolute alignment term and a scale-shift adjusted term [S1, unofficial]:
$$ L_{MDE} = \sum_{I_i \in S} \sum_{p \in P} \left( \| \text{sg}(\bar{D}_i(p)) - D^*_i(p) \| + \| (w_i \cdot \text{sg}(\bar{D}_i(p)) + q_i) - D^*_i(p) \| \right) $$

The regularization loss $L_{reg}$ preserves the local smoothness of the MDE adaptation by comparing the adapted initial depth to the current depth [S1, unofficial]:
$$ L_{reg} = \sum_{I_i \in S} \sum_{p \in P} \| (w_i \cdot \text{sg}(D^*_{i,init}(p)) + q_i) - D^*_i(p) \| $$

If confidence modeling is employed, the losses for seen views $L_{seen}$ and unseen views $L_{unseen}$ are redefined as [S1, unofficial]:
$$ L_{seen} = \sum_{I_i \in S} \sum_{p \in P} M_i(p) \| (w_i \cdot \text{sg}(D^*_i(p)) + q_i) - \bar{D}_i(p) \| $$
$$ L_{unseen} = \sum_{I_l \in U} \sum_{p \in P} M_l(p) \| (w_l \cdot \text{sg}(\bar{D}^*_l(p)) + q_l) - \bar{D}_l(p) \| $$
And the MDE loss becomes [S1, unofficial]:
$$ L_{MDE} = \sum_{I_i \in S} \sum_{p \in P} M_i(p) \left( \| \text{sg}(\bar{D}_i(p)) - \bar{D}^*_i(p) \| + \| (w_i \cdot \text{sg}(\bar{D}_i(p)) + q_i) - \bar{D}^*_i(p) \| \right) $$

**Pitfalls:**
*   **Error Propagation:** Errors in either the NeRF or the MDE can propagate and intensify during distillation if not checked [S1, unofficial]. Confidence modeling is required to mitigate this [S1, unofficial].
*   **Ambiguity:** Without NeRF guidance, MDE cannot resolve convexity/concavity or flat surface orientation [S1, unofficial].

## Subtopics

### Overall Training with Confidence
This variant incorporates confidence masks $M(p)$ into the loss functions to weight the depth consistency losses, preventing error propagation from inaccurate depths [S1, unofficial]. It trains the NeRF and MDE simultaneously, where the MDE provides geometric priors to the NeRF and the NeRF provides absolute geometry to the MDE [S1, unofficial]. The losses $L_{seen}$, $L_{unseen}$, and $L_{MDE}$ are all weighted by their respective confidence maps $M_i(p)$ or $M_l(p)$ [S1, unofficial].

## Operating characteristics
*   **Complexity:** Time complexity is dominated by the iterative rendering of NeRF depths and forward passes of the MDE network for each training step. Memory complexity includes storing the NeRF parameters, MDE parameters, and intermediate depth maps.
*   **Performance:** On ScanNet with 9-10 views, this approach (specifically the DaRF implementation) achieves a PSNR of 16.01 and SSIM of 0.618 [S1, unofficial]. On Tanks and Temples with 10 views, it achieves a PSNR of 18.93 and SSIM of 0.676 [S1, unofficial].
*   **Strengths:** Resolves scale and shift ambiguities of MDE; improves NeRF reconstruction in sparse view settings by providing strong geometric priors; adapts MDE to scene-specific geometry.
*   **Limitations:** Requires simultaneous training of two complex models; sensitive to initialization and confidence modeling accuracy; computationally intensive due to dual optimization loops.
*   **When to use:** Sparse view reconstruction where monocular depth priors are beneficial but ambiguous; scenarios requiring scene-specific MDE adaptation.
*   **When not to use:** Dense view settings where NeRF can reconstruct geometry accurately without priors; real-time applications where dual-network training latency is prohibitive.

## Background
The method was introduced in the paper "DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation" [S1, unofficial]. It addresses the limitations of standard NeRF in sparse view scenarios by leveraging the geometric priors of pre-trained MDE models, while overcoming the MDE's lack of scene-specific absolute depth awareness through bidirectional distillation. The approach builds on the principles of differentiable volume rendering and self-training confidence modeling [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
