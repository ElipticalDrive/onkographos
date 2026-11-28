# Multi-stage Training Protocol

Also called: Pretrain and Joint Train Strategy

## Overview
Multi-stage training protocols decompose complex inverse rendering or radiance field optimization into sequential phases to stabilize convergence and ensure logical dependency between network components. Unlike single-stage end-to-end training, which can suffer from gradient conflicts or poor initialization in coupled systems, this method isolates the learning of geometric priors (albedo, normals, depth) before introducing appearance or lighting variables. It is distinct from standard NeRF training by enforcing a strict hierarchy of feature learning, ensuring that coarse geometry is established before high-frequency details or complex lighting models are optimized.

## How it works
The protocol consists of a sequence of optimization phases. The specific steps depend on the architectural variant (e.g., inverse rendering vs. UV-volume human modeling), but the general logic follows a "Pretrain -> Joint -> Refine" pattern.

### Inputs and Outputs
- **Inputs**: Training images (LDR or HDR), camera poses, and optionally semantic or geometric priors (e.g., SMPL bounds, visible surface volumes).
- **Outputs**: Optimized network weights for all modules (Direct Prediction, Lighting, Re-prediction, UV volumes, NTS) and the final rendered images or inverse rendering results (albedo, normals, depth, lighting).

### Data Structures
The data structures vary by variant.
1.  **Inverse Rendering Variant [S1]**:
    -   **Direct Prediction Module ($h_{DP}$)**: Predicts intermediate outputs $(\tilde{A}, \tilde{N}, \tilde{D})$ for albedo, normal, and depth.
    -   **Lighting Joint Prediction Module**: Predicts lighting $\hat{L}$ based on inputs from $h_{DP}$.
    -   **Joint Re-prediction Module**: Refines $(\tilde{A}, \tilde{N}, \tilde{D})$.
    -   **Final Outputs**: Joint predictions $(\hat{A}, \hat{N}, \hat{D}, \hat{L})$ and intermediate outputs $(\tilde{A}, \tilde{N}, \tilde{D})$.
2.  **UV Volume Variant [S2]**:
    -   **UV Volumes**: Coarse geometry representation.
    -   **Neural Texture Stack (NTS)**: High-frequency detail representation.
    -   **Ray Sampling**: 64 points sampled along each camera ray between near/far bounds estimated by SMPL.

### Numbered Steps
The following steps describe the general progression, with specific details for each variant in the Subtopics section.

1.  **Pretrain Geometric Priors**: Train the modules responsible for coarse geometry (albedo, normal, depth) independently or with a subset of the loss. This ensures reasonable initial values for subsequent modules that depend on these properties [S1].
2.  **Joint Train Core Modules**: Unfreeze and jointly train the geometric modules and the primary appearance/lighting modules using a multi-task loss function [S1].
3.  **Refine/Re-predict**: Freeze the weights of the previously trained modules and train a dedicated refinement module (e.g., Joint Re-prediction) to correct errors in the geometric outputs [S1].
4.  **End-to-End Finetune**: Jointly finetune all modules end-to-end using the full multi-task loss on both intermediate and final outputs to resolve any inconsistencies introduced by freezing [S1].

### Formulas
**Multi-task Loss for Inverse Rendering [S1]**:
$$ L = \lambda_A L_{albedo} + \lambda_N L_{normal} + \lambda_D L_{depth} + \lambda_L L_{light} + \lambda_{visible} L_{visible} + \lambda_{reg} L_{reg} + \lambda_{rerender} L_{rerender} $$
Where $\lambda$ are hyperparameters balancing the different loss terms.

**Learning Rate Decay for UV Volume Variant [S2]**:
The learning rates for UV volumes and NTS decay exponentially over epochs:
$$ l_{uv} = 1 \times 10^{-3} \gamma_{250}^{\text{epoch}} $$
$$ l_{nts} = 5 \times 10^{-4} \gamma_{1000}^{\text{epoch}} $$
Where $\gamma = 1 \times 10^{-1}$ is the decay rate. Note: The sources specify different base epochs for the decay start or normalization ($250$ vs $1000$) in the exponents, implying a piecewise or distinct decay schedule for each branch [S2].

### Parameters
-   **Optimizer**: Adam [S1, S2].
-   **Ray Sampling**: 64 points per ray [S2].
-   **Volume Resolution**: $128^3$ for initialized visible surface volume and predicted VSG lighting [S1].
-   **Hardware**: Single NVIDIA A100 GPU [S2].
-   **Training Duration**: ~200k iterations (~20 hours) for 26-view videos of 100 frames at 960x540 resolution [S2].

### Edge Cases and Pitfalls
-   **Dependency Violation**: In the inverse rendering variant, the Lighting Joint Prediction Module depends on the albedo/normal/depth outputs. Training them simultaneously from scratch can lead to poor convergence if the geometric priors are not reasonable [S1].
-   **UV Volume Warm-start**: In the UV volume variant, NTS optimization is ineffective if UV volumes are not warmed up first, as NTS relies on a coarse geometry to fit high-frequency signals [S2].
-   **Precision**: The sources do not specify floating-point precision requirements, but standard practice suggests FP32 for training stability [G].

## Subtopics

### [MobileNeRF Staged Training](15503-mobilenerf-staged-training.md)
MobileNeRF Staged Training is a three-stage optimization protocol that sequentially trains the geometry, acceleration structures, and appearance networks of the MobileNeRF representation. It decouples joint optimization into phases with adaptive ray marching to stabilize convergence.

### [Multi-resolution Feature Grid Pre-train](17802-multi-resolution-feature-grid-pre-train.md)
Multi-resolution Feature Grid Pre-train is a method that constructs a multi-resolution ground feature plane representation for large urban scenes, capturing density and appearance separately to reduce memory footprint and guide subsequent NeRF sampling. It differs from standard NeRF training by decoupling the initial scene representation into a compact, low-rank tensor factorization (the grid branch) that is pre-trained before joint optimization with a full 3D NeRF branch. This approach leverages the planar nature of urban environments to achieve robust performance with sparse view data.

### UV Volume Training Strategy
The UV Volume Training Strategy uses two distinct optimization strategies for UV volumes and the Neural Texture Stack (NTS) branch [S2]. UV volumes are learned primarily in early stages to warm-start coarse geometry, while NTS is optimized later to fit high-frequency signals [S2]. The learning rates decay exponentially: $l_{uv} = 1 \times 10^{-3} \gamma_{250}^{\text{epoch}}$ and $l_{nts} = 5 \times 10^{-4} \gamma_{1000}^{\text{epoch}}$, with $\gamma = 1 \times 10^{-1}$ [S2]. Ray sampling uses 64 points along camera rays between SMPL-estimated bounds [S2]. Training takes approximately 20 hours (200k iterations) on a single NVIDIA A100 GPU for 26-view videos of 100 frames at 960x540 resolution [S2].

## Operating characteristics
-   **Time Complexity**: The multi-stage approach increases total training time due to sequential phases, but reduces the risk of divergence and the need for extensive hyperparameter tuning for joint optimization [S1, S2].
-   **Memory Complexity**: Depends on the variant. The inverse rendering variant uses a $128^3$ volume [S1]. The UV volume variant uses UV volumes and NTS, with memory optimized for real-time rendering [S2].
-   **Strengths**: Stabilizes convergence for complex, coupled models (e.g., lighting + geometry); ensures geometric priors are reasonable before appearance optimization; allows for specialized learning rates for different components.
-   **Limitations**: Requires careful scheduling of when to freeze/unfreeze modules; may not be necessary for simpler radiance field models that do not have strong inter-component dependencies.
-   **When to Use**: When training inverse rendering models with separate lighting and geometry predictors, or when using UV-based representations where coarse geometry must precede fine detail.
-   **When Not to Use**: For standard NeRF training without explicit inverse rendering components or UV mappings, where end-to-end training is sufficient.

## Background
The Multi-stage Training Protocol originates from research in inverse rendering and human performance modeling. The inverse rendering variant was introduced in "Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting" (2021) to handle the coupling between albedo, normal, depth, and lighting predictions [S1]. The UV volume variant was introduced in "UV Volumes for Real-time Rendering of Editable Free-view Human Performance" (2023) to address the challenges of rendering editable human performances with high-frequency details [S2]. These methods address the instability of jointly optimizing geometric and photometric properties in complex neural representations.

## Sources
[S1] Learning Indoor Inverse Rendering with 3D Spatially-Varying Lighting, https://doi.org/10.1109/iccv48922.2021.01231, fetched 2026-10-26
[S2] UV Volumes for Real-time Rendering of Editable Free-view Human Performance, https://doi.org/10.1109/cvpr52729.2023.01595, fetched 2026-10-26
