# FineSplat

Also called: Detailed Splats, Fine Patches, Local Groups

## Overview
FineSplat is a feed-forward 3D Gaussian Splatting (3DGS) framework that predicts a set of 3D Gaussians from $N$ context views (with or without known poses) using a two-stage dual-grouping strategy to balance local detail preservation with global multi-view reasoning [S1, unofficial]. It distinguishes itself from sibling methods like [SparSplat](15813-sparsplat.md) and [Generalizable Pixel-Wise 3D Gaussian Splatting](17059-generalizable-pixel-wise-3d-gaussian-splatting.md) by employing high-resolution $4 \times 4$ patch tokens and serialized 3D spatial attention, avoiding the prohibitive cost of global multi-view attention while preserving fine-grained visual information [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** $N$ context views (images $I \in \mathbb{R}^{N \times H \times W \times 3}$) and their camera poses (intrinsics and extrinsics). Poses may be known or predicted.
*   **Output:** A set of 3D Gaussian primitives (centers, covariances, colors, opacities) representing the scene.

**Data Structures:**
*   **Tokens:** A set of $M$ fine-grained tokens $z \in \mathbb{R}^{M \times d}$, where $M = N \cdot (H/p) \cdot (W/p)$ and $p=4$ is the patch size.
*   **Gaussians:** $K$ Gaussian primitives predicted per token.

**Parameters:**
*   $p$: Patch size, fixed at $4 \times 4$ pixels [S1, unofficial].
*   $T$: Number of iterative refinement steps in Stage 2, fixed at $4$ [S1, unofficial].
*   $K$: Number of Gaussians per token (typically 4) [S1, unofficial].
*   $s$: Sub-patch stride, fixed at $2$ [S1, unofficial].
*   $S$: Group size for serialized attention (number of consecutive points along the space-filling curve) [S1, unofficial].
*   $N$: Number of input context views. Training samples $N \in \{6, 12, 24, 32\}$ [S1, unofficial].

**Algorithm:**

1.  **Scene Normalization:**
    To handle scale ambiguity in feed-forward reconstruction, normalize scene scale using context camera poses [S1, unofficial]. Express poses in the mean context camera frame. Rescale translations so that all context camera centers lie within $[-1, 1]^3$. Apply the same transform to target poses [S1, unofficial].

2.  **Stage 1: View-Wise Token Grouping:**
    a.  **Feature Extraction:** Extract feature maps $F^l$ for levels $l \in \{1, 2, 3\}$ (finest to coarsest) from the input images using a backbone (e.g., MVP architecture) [S1, unofficial].
    b.  **Token Initialization:** Fuse features into $M$ fine-grained tokens $z^{(0)}$ using Equation (1) [S1, unofficial]:
        $$ z^{(0)} = \text{fuse}(\text{up}(\text{fuse}(\text{up}(F^3) + F^2)) + F^1) $$
        where $\text{up}(\cdot)$ is bilinear upsampling and $\text{fuse}(\cdot)$ is a convolutional fusion block. $M = N \cdot (H/p) \cdot (W/p)$ [S1, unofficial].
    c.  **View-Wise Attention:** Encode tokens using view-wise token grouping. Perform fine-grained attention within small view groups and exchange information across all views via coarser global tokens [S1, unofficial]. This restricts attention scope to keep multi-view reasoning affordable [S1, unofficial].

3.  **Stage 2: 3D Spatial Token Grouping (Iterative Refinement for $t = 1 \dots T$):**
    a.  **Gaussian Prediction:** Decode tokens $z^{(t-1)}$ into an initial set of Gaussians using a lightweight Gaussian head $\mathcal{D}$ (linear decoder). For each token, predict $K$ Gaussians [S1, unofficial].
    b.  **Patch-Aligned Sub-Pixel Reconstruction:**
        i.  Predict 4 Gaussians per $4 \times 4$ token, assigning each to a non-overlapping $2 \times 2$ sub-patch [S1, unofficial].
        ii. Learn a 2D image-space offset $\Delta = (\Delta_x, \Delta_y)$ for each Gaussian [S1, unofficial].
        iii. Bound the offset to keep the Gaussian within its sub-patch:
            $$ \delta_x = 2s \tanh(\Delta_x), \quad \delta_y = 2s \tanh(\Delta_y) $$
            where $s=2$ is the sub-patch stride. This ensures $\delta_x, \delta_y \in [-s/2, s/2]$ [S1, unofficial].
        iv. Compute displaced coordinate $(u, v) = (u_c, v_c) + \delta$, where $(u_c, v_c)$ is the sub-patch center [S1, unofficial].
        v.  **Ray Unprojection:** Unproject $(u, v)$ using predicted camera intrinsics. Transform and normalize direction using camera-to-world rotation, using camera translation as ray origin [S1, unofficial].
    c.  **Rendering:** Render the predicted Gaussians into every context view to obtain rendered images $\hat{I}_n$ [S1, unofficial].
    d.  **Error Feature Computation:**
        i.  Compute signed error map $E_n = \hat{I}_n - I_n$ for each view $n$ [S1, unofficial].
        ii. Compute feature-level difference $\Delta \phi_n = \phi(\hat{I}_n) - \phi(I_n)$ using a pretrained ResNet-18 encoder $\phi(\cdot)$ [S1, unofficial].
        iii. Combine with finest-scale features $F_n^1$ from Stage 1 via linear embeddings $E_{err}$ and $E_{feat}$:
            $$ F_n^{\text{err}} = [E_{err}(E_n, \Delta \phi_n); E_{\text{feat}}(F_n^1)] \in \mathbb{R}^{(H/p) \times (W/p) \times d_e} $$
            where $[\cdot; \cdot]$ represents concatenation [S1, unofficial].
    e.  **Error Aggregation:**
        i.  For each token, gather two error signals:
            *   Self error feature $e_{\text{self}}$ from the patch where the token is predicted.
            *   Aggregated error feature $e_{\text{agg}}$ averaged over all context views where the token is visible (including views outside the Stage 1 view group) [S1, unofficial].
    f.  **3D Spatial Grouping:**
        i.  Lift tokens to 3D space using predicted Gaussian centers [S1, unofficial].
        ii. Order tokens along a space-filling curve and partition them into groups of $S$ consecutive points [S1, unofficial].
        iii. Use four attention blocks, each using a different space-filling curve, so each token meets a different set of neighbors [S1, unofficial].
        iv. Apply serialized self-attention within each group to exchange error signals [S1, unofficial].
        v.  Update tokens:
            $$ \Delta z_j^{(i)} = \text{Linear}(\text{SerialAttn}(h_j^{(i)})) $$
            $$ z_j^{(i+1)} = z_j^{(i)} + \Delta z_j^{(i)} $$
            where $h_j^{(i)}$ are the features of the token in the 3D group [S1, unofficial].
    g.  **Repeat:** Go to step 3a with updated tokens $z^{(t)}$.

**Pitfalls:**
*   **Blind Spots:** Fixed view grouping in Stage 1 leaves blind spots where fine-grained tokens in different groups cannot interact. Stage 2 3D spatial grouping connects tokens across view groups once lifted to 3D [S1, unofficial].
*   **Scale Ambiguity:** Feed-forward reconstruction is scale-ambiguous. Must normalize scene scale using context camera poses as described in Step 1 [S1, unofficial].
*   **Iteration Count:** Quality improves with more Stage 2 steps, but going beyond $T=4$ yields no further gain [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** FineSplat is faster than explicit k-nearest-neighbor search for 3D grouping and scales favorably with token count due to serialized attention [S1, unofficial]. It is faster than [SparSplat](15813-sparsplat.md) and [Generalizable Pixel-Wise 3D Gaussian Splatting](17059-generalizable-pixel-wise-3d-gaussian-splatting.md) variants using larger patches ($14 \times 14$) for 6 and 12 input views, and remains efficient with 24 views [S1, unofficial].
*   **Memory Complexity:** Uses a small primitive budget (e.g., 62K Gaussians for 6 views on DL3DV) compared to baselines like ZipSplat (1.5M Gaussians) [S1, unofficial].
*   **Performance:**
    *   DL3DV (6 views): 27.325 PSNR, 0.871 SSIM, 0.115 LPIPS (62K Gaussians) [S1, unofficial].
    *   RE10K (6 views): 27.394 PSNR, 0.886 SSIM, 0.116 LPIPS (75K Gaussians) [S1, unofficial].
    *   Mip-NeRF 360 (128 views): 23.463 PSNR, 0.706 SSIM, 0.214 LPIPS (1.6M Gaussians) [S1, unofficial].
*   **Strengths:** High quality with few Gaussians; efficient dual-grouping strategy; robust to varying numbers of input views.
*   **Limitations:** Requires camera poses (known or predicted); performance on Mip-NeRF 360 (zero-shot) is lower than on DL3DV, likely due to domain shift and larger scene scale [S1, unofficial].

## Background
FineSplat was introduced in 2026 by [S1, unofficial]. It builds on [Direct Pixel-Wise Regression](18786-direct-pixel-wise-regression.md) and [Feed-forward Generalizable Reconstruction](16746-feed-forward-generalizable-reconstruction.md) principles. It is trained on the DL3DV dataset and evaluated on DL3DV, RealEstate10K, and Mip-NeRF 360 for novel view synthesis [S1, unofficial].

## Sources
[S1] FineSplat: Fine Patches, Local Groups, Detailed Splats, https://arxiv.org/abs/2610.25011, fetched 2026-10-29 (unofficial)
