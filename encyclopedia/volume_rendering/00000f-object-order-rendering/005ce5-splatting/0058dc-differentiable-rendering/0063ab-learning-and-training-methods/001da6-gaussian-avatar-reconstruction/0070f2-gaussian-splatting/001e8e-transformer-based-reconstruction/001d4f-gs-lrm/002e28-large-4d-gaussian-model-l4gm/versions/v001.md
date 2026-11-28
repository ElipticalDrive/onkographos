# Large 4D Gaussian Model (L4GM)

## Overview
L4GM is a feed-forward 4D large reconstruction model that generates per-frame 3D Gaussian Splatting (3DGS) representations from a single-view video input in approximately one second [S1, unofficial]. It addresses the limitation of previous 4D generation models that were restricted to object-centric scenes by enabling the reconstruction of dynamic content from monocular video [S1, unofficial]. Unlike iterative optimization methods, L4GM performs inference in a single forward pass, utilizing temporal self-attention to upsample the output to higher frame rates for temporal smoothness [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A single-view video sequence.
- Output: Per-frame 3D Gaussian parameters (position, scale, rotation, opacity, spherical harmonics coefficients) for each frame in the video.

**Data Structures:**
The model outputs a set of 3D Gaussians $\mathcal{G}_t$ for each frame $t$. Each Gaussian $i$ is defined by:
- Mean position $\mu_i \in \mathbb{R}^3$
- Scale $s_i \in \mathbb{R}^3$
- Rotation $q_i \in \mathbb{R}^4$ (unit quaternion)
- Opacity $\alpha_i \in [0, 1]$
- Spherical Harmonics coefficients $c_{i, l, m}$ for color approximation.

**Method Steps:**
1. **Video Encoding:** The input video is processed through a transformer-based encoder. The architecture is built upon LGM (Large Gaussian Model), which tokenizes image and camera parameters [S1, unofficial]. For video, the temporal dimension is incorporated, likely via 3D tokenization or sequential processing, though specific architectural details of the video encoder are not explicitly detailed in the sources [S1, unofficial].
2. **Forward Pass:** The encoded tokens are passed through the transformer blocks. The model is designed to output 3DGS parameters directly in one forward pass, taking approximately 1 second [S1, unofficial].
3. **Temporal Self-Attention:** To address frame rate limitations and ensure temporal consistency, temporal self-attention is applied to the output tokens. This step upsamples the generated Gaussians to a higher frame rate, enhancing temporal smoothness [S1, unofficial].
4. **Flow Field Regularization:** A flow field is used for temporal regularization during training to ensure coherent motion between frames [S1, unofficial].

**Parameters:**
- Training Dataset: An Objaverse-derived multiview-video dataset containing 44,000 objects, 12 million videos, and 300 million frames [S1, unofficial].
- Inference Time: ~1 second per video [S1, unofficial].

**Edge Cases and Pitfalls:**
- The method relies on the quality of the input video; poor lighting or occlusions may affect reconstruction quality.
- The temporal self-attention mechanism requires sufficient temporal coherence in the input video to effectively upsample and smooth the output.

## Subtopics

### Comp4D
Comp4D is a method designed to overcome the object-centric limitation of previous 4D generation models [S2, unofficial]. It operates by first generating individual 4D objects and then composing them subject to trajectory constraints [S2, unofficial]. This approach allows for more complex scenes beyond single-object reconstructions.

## Operating characteristics
- **Time Complexity:** Inference is fast, taking approximately 1 second for a single forward pass [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Marked as a 2026 SOTA update for 4D reconstruction [S1, unofficial].
- **Strengths:** Fast, feed-forward inference; handles single-view video input; temporal upsampling via self-attention.
- **Limitations:** Requires a large training dataset; specific architectural details of the video encoder are not provided.
- **When to Use:** When fast 4D reconstruction from monocular video is required.
- **When Not to Use:** When iterative optimization for higher fidelity is acceptable, or when multi-view input is available and not restricted to single-view.

## Background
L4GM was introduced as a 2026 SOTA update in the field of 4D reconstruction [S1, unofficial]. It builds upon LGM and is trained on a large-scale Objaverse-derived multiview-video dataset [S1, unofficial]. The model represents a shift from iterative optimization to feed-forward inference for 4D Gaussian Splatting.

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
