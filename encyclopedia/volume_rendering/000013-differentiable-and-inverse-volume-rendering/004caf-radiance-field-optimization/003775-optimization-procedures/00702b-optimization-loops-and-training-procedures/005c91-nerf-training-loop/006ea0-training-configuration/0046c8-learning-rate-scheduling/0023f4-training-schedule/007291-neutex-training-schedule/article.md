# NeuTex Training Schedule
Also called: NeuTex Training Procedure
## Overview
The NeuTex Training Schedule is a multi-stage optimization procedure for training the NeuTex neural texture mapping model. It solves the instability issues inherent in jointly optimizing geometry, texture, and mapping networks by decoupling the initialization of the UV-mapping components from the final high-fidelity texture synthesis. It sets itself apart from standard NeRF training loops by explicitly prioritizing geometric consistency via Chamfer loss and cycle consistency before transitioning to full volumetric rendering optimization, and by employing a final stage that freezes geometry and mapping networks to refine texture details [S1].
## How it works
The method consists of three sequential training stages.

**Inputs:**
*   A set of input images with known camera poses.
*   A sparse ground-truth point cloud (used in Stage 1).
*   Neural networks: Geometry network $F_\sigma$, Texture mapping network $F_{uv}$, Inverse texture mapping network $F_{uv}^{-1}$, and Texture network $F_{\text{tex}}$.

**Outputs:**
*   Optimized weights for $F_\sigma$, $F_{uv}$, $F_{uv}^{-1}$, and $F_{\text{tex}}$.

**Data Structures:**
*   The networks are standard multi-layer perceptrons (MLPs) or similar differentiable functions. Specific architectural details (layer sizes, activation functions) are not provided in the training schedule notes [S1].
*   Loss functions: Chamfer loss ($L_{\text{Chamfer}}$), Rendering loss ($L_{\text{render}}$), and the full loss defined in Equation 14 of the source ($L_{\text{full}}$) [S1].

**Steps:**

1.  **Stage 1: Geometric Initialization (50,000 iterations)**
    *   Train all networks ($F_\sigma$, $F_{uv}$, $F_{uv}^{-1}$, $F_{\text{tex}}$) jointly.
    *   The total loss is a combination of Chamfer loss and rendering loss:
        $$ L_{\text{stage1}} = L_{\text{Chamfer}} + L_{\text{render}} $$
    *   The Chamfer loss aligns the 3D geometry output with the sparse ground-truth point cloud.
    *   The rendering loss ensures the projected images match the input views.
    *   Run for 50,000 iterations [S1].

2.  **Stage 2: Full Optimization (500,000 iterations)**
    *   Remove the Chamfer loss.
    *   Train all networks jointly using the full loss function (Equation 14 from the source), which includes rendering loss and cycle consistency losses between $F_{uv}$ and $F_{uv}^{-1}$ [S1].
    *   $$ L_{\text{stage2}} = L_{\text{full}} $$
    *   Run until convergence, which takes approximately 500,000 iterations [S1].

3.  **Stage 3: Texture Fine-tuning**
    *   Freeze the weights of the geometry network $F_\sigma$, the texture mapping network $F_{uv}$, and the inverse texture mapping network $F_{uv}^{-1}$.
    *   Continue training only the texture network $F_{\text{tex}}$ until convergence.
    *   This stage focuses on improving high-frequency texture details without disturbing the established geometry and mapping [S1].

**Parameters:**
*   Iterations for Stage 1: 50,000 [S1].
*   Iterations for Stage 2: ~500,000 [S1].
*   Hardware: Single NVIDIA RTX 2080Ti GPU [S1].
*   Total Training Time: 2-3 days [S1].

**Edge Cases and Pitfalls:**
*   The schedule relies on the availability of a sparse ground-truth point cloud for the Chamfer loss in Stage 1. If this is unavailable, the initialization strategy may need adaptation [G].
*   The source does not specify learning rates, batch sizes, or optimizer settings (e.g., Adam parameters). These must be determined by the implementer or inferred from standard practices for similar neural rendering methods [G].
*   The exact formulation of "Equation 14" is not provided in the notes. The implementer must refer to the original paper for the specific weights and terms comprising $L_{\text{full}}$ [S1].

## Subtopics

### [NeuTex Initialization Training Procedure](22567-neutex-initialization-training-procedure.md)
The NeuTex Initialization Training Procedure is a pre-training stage designed to initialize the inverse texture mapping network $F_{uv}^{-1}$ and the texture mapping network $F_{uv}$ before the main optimization loop.

## Operating characteristics
*   **Time Complexity:** The method requires approximately 550,000 iterations of training plus fine-tuning. On a single RTX 2080Ti GPU, this takes 2-3 days [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** The staged approach stabilizes training by first establishing geometric coherence before optimizing texture and mapping consistency. The final fine-tuning stage allows for high-quality texture details without degrading geometry [S1].
*   **Limitations:** The training time is significant (2-3 days per scene). The method requires sparse ground-truth geometry for the initial stage, which may not always be available [S1].
*   **When to use:** When high-fidelity texture and geometry are required for neural texture mapping and sparse ground-truth data is available for initialization.
*   **When not to use:** When training time is constrained or when sparse ground-truth geometry is not available.

## Background
The NeuTex Training Schedule was introduced in the paper "NeuTex: Neural Texture Mapping for Volumetric Neural Rendering" [S1]. The method addresses the challenges of training neural texture mapping networks, which involve complex interactions between geometry, UV mapping, and texture synthesis. The multi-stage schedule was developed to mitigate training instability and improve the final visual quality of the rendered textures [S1].

## Sources
[S1] NeuTex: Neural Texture Mapping for Volumetric Neural Rendering, https://doi.org/10.1109/cvpr46437.2021.00704, fetched 2026-10-26
