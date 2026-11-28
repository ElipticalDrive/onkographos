# Adaptive Gaussian Selector

Also called: GsNet, Gumbel-Softmax Selection

## Overview
The Adaptive Gaussian Selector is a learnable module within the FlexGS framework that adaptively selects a subset of 3D Gaussians during training based on their attributes and a desired elastic ratio $e$, using a differentiable mask to enable gradient-based optimization [S1]. It sets itself apart from non-differentiable thresholding or hard masking by allowing gradients to flow through the selection decision via the Straight-Through Estimator (STE) applied to Gumbel-Softmax outputs, unlike sibling methods that rely on fixed heuristics or non-differentiable hard thresholds [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $A_i$: The set of attributes for the $i$-th Gaussian (concatenated into vector $\xi$).
    *   $e$: The desired elastic ratio (sparsity level), sampled from a set $\mathcal{E}$.
    *   $N$: Total number of Gaussians.
*   **Outputs:**
    *   $\hat{M}_i$: A binary mask value (0 or 1) for each Gaussian, indicating selection.
    *   $z$: Logits for the categorical distribution.

**Data Structures:**
*   **GsNet Parameters:**
    *   $W^{1e} \in \mathbb{R}^D$: Weight matrix for ratio embedding.
    *   $W^{2e} \in \mathbb{R}^{D \times D}$: Weight matrix for ratio embedding.
    *   $W^{1a} \in \mathbb{R}^{H \times D}$: Weight matrix for attribute embedding.
    *   $W^{2a} \in \mathbb{R}^{D \times D}$: Weight matrix for attribute embedding.
    *   $W \in \mathbb{R}^{2D \times 2}$: Mixing weight matrix.
    *   $D$: Hidden feature dimension (typically 64 or similar, not explicitly fixed in notes but implied by matrix shapes) [S1].
    *   $H$: Dimension of concatenated attributes $\xi$ [S1].
*   **Global Importance (GI):**
    *   $GI_i$: Scalar score for each Gaussian, updated every 1k iterations [S1].
    *   $V(s)$: Volume of Gaussian $i$, calculated from scaling vector $s$.
    *   $V_{90\%}$: 90% of the largest volume among all Gaussians [S1].

**Numbered Steps:**

1.  **Feature Embedding:**
    *   Convert the elastic ratio $e$ to a feature vector $h_e$:
        $$ h_e = \sigma(\sigma(e W^{1e}) W^{2e}) $$
        where $\sigma$ is ReLU [S1].
    *   Concatenate Gaussian attributes $A_i$ into $\xi$. Convert $\xi$ to a feature vector $h_a$:
        $$ h_a = \sigma(\sigma(\xi W^{1a}) W^{2a}) $$
        where $\sigma$ is ReLU [S1].

2.  **Logit Generation:**
    *   Concatenate $h_e$ and $h_a$ and project to logits $z \in \mathbb{R}^2$:
        $$ z = [h_e, h_a] W $$
        where $W \in \mathbb{R}^{2D \times 2}$ [S1].

3.  **Differentiable Sampling (Training):**
    *   Sample binary masks $\hat{M}_i$ using Gumbel-Softmax to model the categorical distribution $P(\lambda_i = m | A_i, e)$:
        $$ P(\lambda_i = m | A_i, e) = \frac{\exp((z_m + g_m)/\tau)}{\sum_{m=0,1} \exp((z_m + g_m)/\tau)} $$
        where $g_m \sim \text{Gumbel}(0, 1)$ and $\tau$ is the temperature parameter [S1].
    *   The temperature $\tau$ is exponentially decayed during optimization to converge to a one-hot vector (hard selection) [S1].
    *   [G] The Straight-Through Estimator (STE) is applied: gradients flow through the continuous Gumbel-Softmax output during the backward pass, while the discrete sample is used for the forward pass [S1].

4.  **Global Importance (GI) Calculation:**
    *   Compute $GI_i$ for all Gaussians to guide selection:
        $$ GI_i = \sum_{p=1}^{QHW} \mathbb{1}(G(X_i), r_p) \cdot \gamma(s_i) \cdot \alpha_i \cdot T(i, p) $$
        where:
        *   $Q$: Number of seen views.
        *   $H, W$: Image height and width.
        *   $\mathbb{1}(G(X_i), r_p)$: Overlap indicator (1 if Gaussian $i$'s projection overlaps pixel $p$, 0 otherwise).
        *   $\alpha_i$: Opacity of Gaussian $i$.
        *   $\gamma(s_i) = \max(V(s_i)/V_{90\%}, 1)$, where $V(s) = |s_1 s_2 s_3|\pi/3$ [S1].
        *   $T(i, p) = \prod_{t=1}^{ID(i,p)-1} (1 - \alpha_t)$, where $ID(i,p)$ is the index of Gaussian $i$ among sorted overlapping Gaussians at pixel $p$ [S1].
    *   Update $GI_i$ every 1k iterations to ensure guidance accuracy [S1].

5.  **Guidance Mask Generation:**
    *   Select the top $\lfloor eN \rfloor$ Gaussians with the highest $GI$ scores to form the guidance mask $M_{GI}$ [S1].

6.  **Regularization Losses:**
    *   Calculate the gap between the predicted mask $\hat{M}$ and the guidance mask $M_{GI}$:
        $$ L_{GI} = \|\hat{M} - M_{GI}\| $$
        [S1].
    *   Calculate the sparsity ratio constraint:
        $$ L_{spar} = e - \sum_{m \in \hat{M}} \frac{m}{N} $$
        [S1].

7.  **Opacity Update and Rendering:**
    *   Update opacity of all Gaussians:
        $$ \alpha_i = \alpha_i \cdot \hat{M}_i $$
        This renders unselected Gaussians transparent [S1].
    *   Render two images:
        *   $I_{es}$: Rendered image using only selected Gaussians (based on $\hat{M}$).
        *   $I_f$: Rendered image using all Gaussians (baseline).
    *   Compute rendering loss against ground truth $I_{GT}$:
        $$ L_{render} = \|I_{es} - I_{GT}\| + \|I_f - I_{GT}\| $$
        [S1].

8.  **Total Loss:**
    *   Combine losses with coefficients $\beta_1 = 1.0$ and $\beta_2 = 0.01$:
        $$ L = L_{render} + \beta_1 L_{GI} + \beta_2 L_{spar} $$
        [S1].

**Parameters:**
*   $\tau$: Temperature parameter for Gumbel-Softmax. Exponentially decayed during optimization [S1].
*   $\beta_1$: Weight for GI guidance regularization. Value: 1.0 [S1].
*   $\beta_2$: Weight for ratio constraint regularization. Value: 0.01 [S1].
*   $\mathcal{E}$: Set of elastic ratios sampled during training, e.g., $\{0.01, 0.05, 0.10, 0.15\}$ [S1].

**Edge Cases and Pitfalls:**
*   **Sparsity Supervision Drift:** During training, enforcing sparsity supervision on masks predicted by GsNet does not guarantee exact activation counts at inference (e.g., target 0.20 yields ~19.5%) [S1]. This is addressed in the Elastic Inference Procedure.
*   **Convergence at High Ratios:** GsNet may struggle to converge smoothly at high ratios due to insufficient constraints, as there are numerous ways to mask Gaussians [S1]. The GI guidance mechanism addresses this.
*   **GI Update Frequency:** GI must be updated frequently (every 1k iterations) because Gaussian attributes evolve during training, causing importance scores to change [S1].

## Subtopics

### [Differentiable Gaussian Selection via Gumbel-Softmax and STE](16109-differentiable-gaussian-selection-via-gumbel-softmax-and-ste.md)
This method provides a differentiable mechanism to select a subset of 3D Gaussians during training by converting continuous logits into a binary mask, enabling gradient-based optimization of Gaussian attributes despite the discrete nature of selection. It sets itself apart from non-differentiable thresholding or hard masking by allowing gradients to flow through the selection decision via the Straight-Through Estimator (STE) applied to Gumbel-Softmax outputs, unlike sibling methods that rely on fixed heuristics or non-differentiable hard thresholds.

### Elastic Inference Procedure
An inference-time protocol that discards unselected Gaussians entirely and uses continuous logits to guarantee an exact target sparsity ratio $e$ [S1].
*   **Definition:** Discards unselected Gaussians at inference time [S1].
*   **Steps:**
    1.  Call PyTorch's `F.gumbel_softmax` with `hard=False` to obtain continuous logits instead of discrete masks [S1].
    2.  Select the top $\lfloor eN \rfloor$ logits out of $N$ total Gaussians to activate, ensuring the exact target ratio $e$ is met despite sparsity supervision drift [S1].
*   **Pitfall:** Sparsity supervision during training does not guarantee exact activation counts at inference (e.g., target 0.20 yields ~19.5%). The top-k selection on continuous logits fixes this [S1].

## Operating characteristics
*   **Time Complexity:** The selection process involves forward passes through GsNet (lightweight MLPs) and GI calculation. GI calculation requires iterating over all pixels and overlapping Gaussians, which is expensive. GI is updated every 1k iterations to mitigate this [S1].
*   **Memory Complexity:** Requires storing GsNet parameters ($W^{1e}, W^{2e}, W^{1a}, W^{2a}, W$) and Global Importance scores for all Gaussians [S1].
*   **Measured Performance:** On Mip-NeRF360 dataset, FlexGS (using this selector) achieves higher PSNR/SSIM than LightGS, C3DGS, and EAGLES at various elastic ratios (1%, 5%, 10%, 15%) [S1]. For example, at 1% ratio, PSNR is 14.455 vs 22.265 for LightGS* (no fine-tuning) or 21.761 for LightGS [S1]. Experiments conducted on NVIDIA A100 GPU [S1].
*   **Scaling:** The method is designed to handle various elastic ratios, allowing flexible deployment [S1].
*   **Strengths:** Differentiable selection allows joint optimization of Gaussian attributes and selection masks. GI guidance ensures stable convergence.
*   **Limitations:** Requires frequent GI updates. Training time may increase due to the additional loss terms and GI calculation.
*   **When to use:** When flexible deployment at different sparsity levels is required, and differentiable optimization of Gaussian selection is needed.
*   **When not to use:** When strict real-time inference with fixed sparsity is required without the overhead of training a selector, or when computational resources for frequent GI updates are limited.

## Background
The Adaptive Gaussian Selector is part of the FlexGS method, introduced in "FlexGS: Train Once, Deploy Everywhere with Many-in-One Flexible 3D Gaussian Splatting" [S1]. It addresses the limitation of fixed heuristic selection rules which may fail to generalize in extreme cases (e.g., selecting 1% of Gaussians) and the need for importance scores to be frequently re-computed as Gaussian attributes evolve during training [S1]. The method builds on 3D Gaussian Splatting (3DGS) [18] and utilizes Gumbel-Softmax [16] for differentiable sampling.

## Sources
[S1] FlexGS: Train Once, Deploy Everywhere with Many-in-One Flexible 3D Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.01523, fetched 2026-10-29
