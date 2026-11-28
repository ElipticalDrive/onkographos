# LoRA Fine-Tuning of DINO

Also called: LoRA-NeRF, NeRF with LoRA-fine-tuned DINO features

## Overview
LoRA Fine-Tuning of DINO adapts frozen DINOv2 features to the geometric objectives of NeRF by applying Low-Rank Adaptation (LoRA) to the attention layers of the Vision Transformer (ViT) backbone [S1, unofficial]. It addresses the misalignment between semantic features and geometric reconstruction by making only low-rank matrices trainable while keeping pre-trained weights frozen [S1, unofficial]. Unlike frozen feature baselines, it allows the model to adapt DINO features to few-shot 3D reconstruction requirements [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Pre-trained DINOv2-base model weights $W_0$, training images, camera poses, and scene geometry.
*   **Output:** Fine-tuned DINO feature extractor with LoRA matrices $(A, B)$, and the NeRF field $F_\Theta$ parameters.

**Data Structures:**
*   **LoRA Matrices:**
    *   $A \in \mathbb{R}^{r \times d}$: Down-projection matrix.
    *   $B \in \mathbb{R}^{d \times r}$: Up-projection matrix.
    *   $r$: LoRA rank (default 16) [S1, unofficial].
    *   $d$: Dimension of the attention layer weights [S1, unofficial].
*   **Feature Projection:** DINO features (768-dim) are projected to 256-dim to match the NeRF MLP hidden dimension [S1, unofficial].

**Steps:**
1.  **Initialize LoRA:** For each attention layer in the DINOv2 ViT backbone, initialize trainable matrices $A$ and $B$. The pre-trained weights $W_0$ remain frozen [S1, unofficial].
2.  **Forward Pass (Feature Extraction):** Compute the attention output using the adapted weight matrix:
    $$
    W = W_0 + BA
    $$
    where $W$ is the effective weight matrix [S1, unofficial]. Extract features $\mathbf{f}(\mathbf{x})$ from the ViT [S1, unofficial].
3.  **Project Features:** Project $\mathbf{f}(\mathbf{x})$ from 768 dimensions to 256 dimensions [S1, unofficial].
4.  **NeRF Forward Pass:** Feed positional encodings $\gamma(\mathbf{x})$, $\gamma(\mathbf{d})$, and projected features $\mathbf{f}'(\mathbf{x})$ into the NeRF MLP $F_\Theta$ [S1, unofficial]:
    $$
    F_\Theta([\gamma(\mathbf{x}), \gamma(\mathbf{d}), \mathbf{f}'(\mathbf{x})]) \rightarrow (\sigma, \mathbf{c})
    $$
5.  **Optimization:** Update only $A$ and $B$ (and NeRF parameters) using the Adam optimizer ($lr = 2 \times 10^{-4}$) for 200 epochs with cosine annealing [S1, unofficial]. Gradient clipping (max norm 1.0) and dropout ($p=0.1$) are applied [S1, unofficial].

**Parameters:**
*   LoRA rank $r = 16$ [S1, unofficial].
*   Optimizer: Adam [S1, unofficial].
*   Learning Rate: $2 \times 10^{-4}$ [S1, unofficial].
*   Epochs: 200 [S1, unofficial].

**Edge Cases:**
*   LoRA is applied selectively to attention layers only [S1, unofficial].
*   DINO features are computed once and cached for efficiency [S1, unofficial].

## Subtopics

### LoRA Fine-Tuning
A technique that fine-tunes the diffusion model with minimal computational overhead, making it adaptable to new tasks by modifying a few parameters within the model [S2].

## Operating characteristics
*   **Time Complexity:** Training typically takes 2-4 hours per experiment [S1, unofficial].
*   **Memory:** Mixed-precision training is used to accelerate experiments [S1, unofficial].
*   **Performance:** On the Lego dataset with 5 training views, LoRA-NeRF achieves a PSNR of 12.97, SSIM of 0.45, and LPIPS of 0.54 [S1, unofficial]. This is slightly worse than Baseline NeRF (PSNR 14.71) but comparable to Frozen DINO-NeRF (PSNR 12.99) [S1, unofficial].
*   **Strengths:** Parameter-efficient adaptation of semantic features to geometric tasks [S1, unofficial].
*   **Limitations:** All DINO variants performed worse than Baseline NeRF on the tested dataset [S1, unofficial].

## Background
Originated in the study of limitations of NeRF with pre-trained vision features for few-shot 3D reconstruction [S1, unofficial]. It leverages LoRA [10] to adapt DINOv2-base features [S1, unofficial]. The implementation is based on PyTorch [27] and uses Weights & Biases [28] for tracking [S1, unofficial].

## Sources
[S1] Limitations of NERF with pre-trained Vision Features for Few-Shot 3D Reconstruction, https://arxiv.org/abs/2506.18208v1, fetched 2026-10-26 (unofficial)
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
