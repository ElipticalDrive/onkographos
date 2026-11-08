# GS-LRM

## Overview
GS-LRM is a feed-forward transformer-based method that reconstructs 3D Gaussian Splatting (3DGS) representations directly from a small set of posed 2D images [S1, unofficial]. It solves the problem of sparse-view 3D reconstruction by tokenizing image and camera data, processing them through a transformer, and decoding the output into full Gaussian parameters (mean, covariance, color, opacity) [S1, unofficial]. It distinguishes itself from sibling methods like [Flash3D](1481-flash3d.md) and [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md) by its specific tokenization strategy for multi-view inputs and direct Gaussian parameter prediction without iterative optimization [S1, unofficial].

## How it works
**Inputs:** A set of $N$ input images (typically $N=2$ to $4$) and their corresponding camera parameters (intrinsics and extrinsics) [S1, unofficial].
**Outputs:** A set of 3D Gaussians defined by mean position $\mu$, covariance matrix $\Sigma$ (or scale/rotation), color $c$, and opacity $\alpha$ [S1, unofficial].

**Data Structures:**
1.  **Image Tokens:** Each input image is processed by a vision encoder to produce a sequence of feature tokens [G].
2.  **Camera Tokens:** Camera parameters are encoded into token embeddings [G].
3.  **Concatenated Sequence:** Image tokens and camera tokens are concatenated into a single sequence [S1, unofficial].

**Steps:**
1.  **Tokenization:** Encode the $N$ input images into image feature tokens [S1, unofficial].
2.  **Pose Encoding:** Encode the corresponding camera poses into camera parameter tokens [S1, unofficial].
3.  **Concatenation:** Concatenate the image tokens and camera tokens into a single input sequence for the transformer [S1, unofficial].
4.  **Transformer Processing:** Pass the concatenated sequence through a transformer block (or stack of blocks) that attends to relationships between image features and camera poses [S1, unofficial].
5.  **Decoding:** Decode the output tokens from the transformer into the full set of 3D Gaussian parameters ($\mu, \Sigma, c, \alpha$) [S1, unofficial].

**Parameters:**
*   Number of input views: Typically 2 to 4 [S1, unofficial].

**Edge Cases:**
*   **View Limitation:** The method is limited to processing 2–4 input views simultaneously [S1, unofficial].
*   **Pose Requirement:** The method requires known camera poses; methods without known poses require additional mechanisms like intrinsics token embeddings [G].

## Subtopics

### Large 4D Gaussian Model (L4GM)
L4GM is a feed-forward 4D large reconstruction model that outputs per-frame 3DGS from a single-view video in one ~1s forward pass [S2, unofficial]. It upsamples to higher fps via temporal self-attention for temporal smoothness [S2, unofficial]. It is built on LGM and trained on a new Objaverse-derived multiview-video dataset containing 44K objects, 12M videos, and 300M frames [S2, unofficial]. It is marked as a 2026 SOTA update [S2, unofficial].

## Operating characteristics
**Time Complexity:** Feed-forward inference, typically ~1s for L4GM on video [S2, unofficial].
**Memory Complexity:** Not given in the sources.
**Strengths:** Fast, non-iterative reconstruction; handles sparse views; direct Gaussian output [S1, unofficial].
**Limitations:** Limited to a small number of input views (2–4) [S1, unofficial].
**When to use:** Sparse-view 3D reconstruction from posed images [S1, unofficial].
**When not to use:** When camera poses are unknown or when iterative optimization is acceptable for higher quality [G].

## Background
GS-LRM was introduced as part of the evolution of feed-forward 3D reconstruction models [S1, unofficial]. It builds on earlier methods like PixelSplat and MVSplat, which also predict Gaussians aligned along camera-to-pixel rays [S1, unofficial]. L4GM, a variant, is noted as a 2026 SOTA update [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
