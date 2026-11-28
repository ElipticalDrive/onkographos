# Large 4D Gaussian Model (L4GM)
## Overview
L4GM is a feed-forward 4D large reconstruction model that generates per-frame 3D Gaussian Splatting (3DGS) representations from a single-view video input in approximately one second [S1, unofficial]. It utilizes temporal self-attention to upsample the output to higher frame rates for temporal smoothness [S1, unofficial]. Built on LGM, it differs from iterative optimization methods by performing inference in a single forward pass [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A single-view video sequence [S1, unofficial].
- Output: Per-frame 3DGS parameters for each frame in the video [S1, unofficial].
**Data Structures:**
Each frame $t$ outputs a set of 3D Gaussians $\mathcal{G}_t$. Each Gaussian $i$ is defined by mean position $\mu_i \in \mathbb{R}^3$, scale $s_i \in \mathbb{R}^3$, rotation $q_i \in \mathbb{R}^4$ (unit quaternion), opacity $\alpha_i \in [0, 1]$, and spherical harmonics coefficients $c_{i, l, m}$ for color approximation [G].
**Method Steps:**
1. **Video Encoding:** The input video is processed through a transformer-based encoder built upon LGM [S1, unofficial]. Specific architectural details of the video encoder are not provided in the sources [S1, unofficial].
2. **Forward Pass:** The encoded tokens pass through transformer blocks to predict 3DGS parameters directly in one forward pass, taking approximately 1 second [S1, unofficial].
3. **Temporal Self-Attention:** Temporal self-attention is applied to the output tokens to upsample the generated Gaussians to a higher frame rate, ensuring temporal smoothness [S1, unofficial].
4. **Training Regularization:** A flow field is used for temporal regularization during training to enforce coherent motion between frames [S1, unofficial].
**Parameters:**
- Training Dataset: Objaverse-derived multiview-video dataset containing 44,000 objects, 12 million videos, and 300 million frames [S1, unofficial].
- Inference Time: ~1 second per video [S1, unofficial].
**Edge Cases and Pitfalls:**
Not given in the sources.
## Subtopics

### Comp4D
Comp4D is a method designed to overcome the object-centric limitation of previous 4D generation models [S2, unofficial]. It first generates individual 4D objects and later composes them subject to trajectory constraints [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Inference takes approximately 1 second per video [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Marked as a 2026 SOTA update for 4D reconstruction [S1, unofficial].
- **Strengths:** Fast feed-forward inference; handles single-view video input; temporal upsampling via self-attention [S1, unofficial].
- **Limitations:** Requires a large training dataset; specific architectural details of the video encoder are not provided [S1, unofficial].
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.
## Background
L4GM was introduced as a 2026 SOTA update in the field of 4D reconstruction [S1, unofficial]. It builds upon LGM and is trained on a large-scale Objaverse-derived multiview-video dataset [S1, unofficial]. The model represents a shift from iterative optimization to feed-forward inference for 4D Gaussian Splatting [S1, unofficial].
## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
