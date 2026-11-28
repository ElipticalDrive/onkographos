# GS-LRM

Also called: Gaussian Large Reconstruction Model

## Overview
GS-LRM is a feed-forward transformer-based method that predicts 3D Gaussian Splatting (3DGS) parameters directly from a small set of input images [S1, unofficial]. It solves the problem of sparse-view reconstruction by tokenizing images and camera poses, processing them through a transformer, and decoding the output into Gaussian attributes without iterative optimization [S1, unofficial]. It distinguishes itself from sibling methods like [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md) by specifically targeting Gaussian parameter prediction rather than mesh or NeRF fields, and from [Flash3D](1481-flash3d.md) by its explicit handling of multi-view camera pose tokens [S1, unofficial].

## How it works
**Inputs:** A set of $N$ input images (typically $N=2$ to $4$) and their corresponding camera parameters (intrinsics and extrinsics) [S1, unofficial].
**Outputs:** A set of 3D Gaussian parameters: mean position $\mu$, covariance $\Sigma$ (or rotation $R$ and scale $S$), color $c$, and opacity $\alpha$ [S1, unofficial].

**Data Structures:**
1.  **Image Tokens:** Each input image is encoded into a sequence of feature tokens using a vision backbone (e.g., ViT) [S1, unofficial].
2.  **Camera Tokens:** Camera parameters are encoded into token embeddings [S1, unofficial].
3.  **Concatenated Sequence:** Image tokens and camera tokens are concatenated into a single sequence [S1, unofficial].

**Steps:**
1.  **Tokenization:** Encode the $N$ input images into image feature tokens [S1, unofficial]. Encode the corresponding camera parameters into camera tokens [S1, unofficial].
2.  **Concatenation:** Concatenate the image tokens and camera tokens into a single sequence [S1, unofficial].
3.  **Transformer Processing:** Pass the concatenated sequence through a transformer block (encoder/decoder) to model spatial and geometric relationships [S1, unofficial].
4.  **Decoding:** Decode the output tokens from the transformer into the final set of 3D Gaussian parameters ($\mu, \Sigma, c, \alpha$) [S1, unofficial].

**Training Objective:** The model is supervised using photometric losses (e.g., LPIPS, SSIM) between rendered outputs and ground truth images [S1, unofficial].

**Edge Cases:**
*   **Camera Pose Accuracy:** The method relies on known camera poses. If poses are inaccurate, reconstruction quality degrades [S1, unofficial].
*   **View Overlap:** Performance may vary with the amount of overlap between input views [S1, unofficial].

## Subtopics

### Large 4D Gaussian Model (L4GM)
L4GM is a feed-forward 4D large reconstruction model that outputs per-frame 3DGS from a single-view video in one ~1s forward pass [S2, unofficial]. It upsamples to higher fps for temporal smoothness via temporal self-attention [S2, unofficial]. It is built on LGM and trained on a new Objaverse-derived multiview-video dataset containing 44K objects, 12M videos, and 300M frames [S2, unofficial]. It is marked as a 2026 SOTA update [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Feed-forward inference, typically ~1s for L4GM variants [S2, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** L4GM is noted as a 2026 SOTA update for 4D reconstruction [S2, unofficial].
*   **Strengths:** Fast, feed-forward inference; no iterative optimization required [S1, unofficial].
*   **Limitations:** Requires known camera poses (unlike NoPoSplat) [S1, unofficial].
*   **When to use:** Sparse-view reconstruction where speed is critical and camera poses are known [S1, unofficial].
*   **When not to use:** When camera poses are unknown or inaccurate [S1, unofficial].

## Background
GS-LRM was introduced as part of the evolution of feed-forward 3D reconstruction models, building upon earlier methods like PixelSplat and MVSplat [S1, unofficial]. It is cited in a tutorial on Gaussian Splatting variants [S1, unofficial]. L4GM, a variant, is marked as a 2026 SOTA update [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
