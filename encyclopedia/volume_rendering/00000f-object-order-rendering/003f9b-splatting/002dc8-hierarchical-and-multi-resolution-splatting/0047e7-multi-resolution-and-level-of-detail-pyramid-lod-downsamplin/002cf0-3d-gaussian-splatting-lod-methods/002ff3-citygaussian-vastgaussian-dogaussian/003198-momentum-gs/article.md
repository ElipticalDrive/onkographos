# Momentum-GS


## Overview
Momentum-GS is a block-wise training method for large-scale 3D Gaussian Splatting that uses a momentum-updated teacher model to guide student optimization, improving convergence quality and restoring cross-block spatial consistency [S1, S2, unofficial]. It decouples the number of training blocks from the GPU count and achieves a ~18.7% LPIPS improvement over CityGaussian using far fewer blocks [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Large-scale scene divided into blocks of 3D Gaussians.
*   **Output:** Optimized Gaussian parameters with restored cross-block spatial consistency.

**Data Structures:**
*   **Student Model:** Gaussian parameters for the current block being optimized.
*   **Teacher Model:** Exponential moving average (EMA) of the parameters, providing global self-distillation guidance [S1, S2, unofficial].
*   **Block Weights:** Scalars adjusting each block based on reconstruction accuracy [S2, unofficial].

**Algorithm Steps:**
1. **Initialization:** Initialize student Gaussian parameters for each block. Initialize the teacher model as an EMA of the student parameters [G].
2. **Block Training:** For each block, optimize the student parameters using a self-distillation loss guided by the teacher model [S2, unofficial].
3. **Block Weighting:** Adjust each block's contribution by its reconstruction accuracy [S2, unofficial].
4. **Teacher Update:** Update the teacher model using an exponential moving average of the student parameters [S1, S2, unofficial].
5. **Iteration:** Repeat until convergence.

**Formulas:**
*   **EMA Update:** $\theta_{\text{teacher}} \leftarrow \alpha \theta_{\text{teacher}} + (1-\alpha) \theta_{\text{student}}$, where $\alpha$ is the momentum coefficient [G].
*   **Block Weighting:** $w_b = f(\text{reconstruction accuracy}_b)$, where the exact function is not specified in the sources [S2, unofficial].

**Parameters:**
*   $\alpha$: Momentum coefficient. Not given in the sources.
*   $\lambda$: Distillation loss weight. Not given in the sources.
*   Block size/overlap: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** EMA updates may require careful handling in mixed-precision training [G].
*   **Boundaries:** Cross-block consistency is critical; improper handling can cause visible artifacts [S2, unofficial].
*   **Degenerate Input:** Blocks with very low reconstruction accuracy may require weight capping to prevent destabilizing the teacher model [G].

## Subtopics

### Distributed Learning for Large-Scale Scenes
Reconstructs large-scale scenes by dividing the problem into distributed parts [S3]. Momentum-GS builds on this approach by adding momentum self-distillation to ensure consistency across distributed blocks [S2, unofficial][S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Requires storing both student and teacher parameters [G].
*   **Measured Performance:** Reports ~18.7% LPIPS improvement over CityGaussian with far fewer blocks [S2, unofficial].
*   **Strengths:** Decouples block count from GPU count, restores cross-block spatial consistency, improves convergence quality [S2, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to Use:** Large-scale scene reconstruction where spatial consistency and convergence quality are critical [S2, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
Momentum-GS was introduced as a 2026 SOTA update for large-scene Gaussian Splatting training [S2, unofficial]. It addresses distributed learning challenges in large-scale scenes, such as geometry accuracy and reconstruction time bottlenecks [S3]. The method leverages momentum self-distillation, where a teacher model guides student optimization to improve convergence quality [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
