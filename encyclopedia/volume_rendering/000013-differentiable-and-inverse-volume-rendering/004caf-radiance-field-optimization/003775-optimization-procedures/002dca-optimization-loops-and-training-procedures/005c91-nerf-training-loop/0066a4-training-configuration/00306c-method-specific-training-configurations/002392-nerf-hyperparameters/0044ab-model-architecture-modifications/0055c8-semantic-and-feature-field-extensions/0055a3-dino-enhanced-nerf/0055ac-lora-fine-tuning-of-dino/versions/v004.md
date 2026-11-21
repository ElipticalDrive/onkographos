# LoRA Fine-Tuning of DINO

Also called: LoRA-NeRF, NeRF with LoRA-fine-tuned DINO features

## Overview
This method adapts frozen DINO vision features to the geometric objectives of NeRF by applying Low-Rank Adaptation (LoRA) to the attention layers of the Vision Transformer (ViT) backbone [S1, unofficial]. It addresses the misalignment between static semantic features and the geometric consistency required for few-shot 3D reconstruction [S1, unofficial]. Unlike frozen feature baselines, this approach allows the model to refine feature representations specifically for the target scene while preserving the pre-trained knowledge [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Pre-trained DINOv2-base model weights ($W_0$), training images, and camera poses.
*   **Output:** Fine-tuned low-rank adaptation matrices ($A, B$) and the frozen base weights ($W_0$). The resulting feature extractor produces features $\mathbf{f}'$ that are fused with NeRF's positional encoding.

**Data Structures:**
*   **Base Weights ($W_0$):** Frozen pre-trained weights from the DINOv2 ViT attention layers. Dimensions are not explicitly specified in the source, but the adaptation matrices define the rank relative to the feature dimension $d$.
*   **Adaptation Matrices:**
    *   $A \in \mathbb{R}^{r \times d}$: Trainable down-projection matrix.
    *   $B \in \mathbb{R}^{d \times r}$: Trainable up-projection matrix.
    *   $r$: LoRA rank, set to 16 [S1, unofficial].
    *   $d$: The dimension of the attention layer input/output (typically 768 for DINOv2-base [S1, unofficial]).

**Numbered Steps:**
1.  **Initialize:** Load the pre-trained DINOv2-base model. Freeze all parameters $W_0$.
2.  **Inject LoRA:** Selectively apply LoRA to the attention layers of the ViT. For each targeted attention weight matrix $W$, replace it with the adapted form:
    $$
    W = W_0 + BA
    $$
    where $W_0$ remains frozen and only $A$ and $B$ are trainable [S1, unofficial].
3.  **Feature Extraction:** Compute DINO features for input images. The model outputs 768-dimensional features [S1, unofficial].
4.  **Projection:** Project the 768-dimensional features to 256 dimensions before fusion with NeRF’s positional encoding [S1, unofficial].
5.  **NeRF Training:** Train the NeRF model using the adapted features. The NeRF function maps $(\mathbf{x}, \mathbf{d})$ to $(\mathbf{c}, \sigma)$ conditioned on the features [S1, unofficial].
6.  **Optimization:** Update only $A$ and $B$ (and NeRF parameters) during training. The DINO backbone weights $W_0$ are not updated [S1, unofficial].

**Parameters:**
*   **LoRA Rank ($r$):** 16 [S1, unofficial].
*   **Feature Dimension:** 768 (input), 256 (output after projection) [S1, unofficial].
*   **Training Time:** Typically 2-4 hours per experiment [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Mixed-precision training is used to accelerate experiments, particularly on Apple Silicon (MPS backend) [S1, unofficial].
*   **Caching:** DINO features are computed once and cached for efficiency [S1, unofficial].
*   **Implementation:** The method is implemented in PyTorch [S1, unofficial].

## Subtopics

### LoRA Fine-Tuning
A technique that fine-tunes the diffusion model with minimal computational overhead, making it adaptable to new tasks by modifying a few parameters within the model [S2].

## Operating characteristics
*   **Performance:** On the Lego dataset with 5 training views, LoRA-NeRF achieves a PSNR of 12.97, SSIM of 0.45, and LPIPS of 0.54 [S1, unofficial]. This is comparable to the frozen DINO-NeRF baseline (PSNR 12.99, SSIM 0.46, LPIPS 0.54) and slightly better than Multi-Scale LoRA-NeRF (PSNR 12.94, SSIM 0.44, LPIPS 0.54) [S1, unofficial].
*   **Comparison:** All DINO variants perform worse than the Baseline NeRF (PSNR 14.71, SSIM 0.46, LPIPS 0.53) on this specific metric set [S1, unofficial].
*   **Efficiency:** The method uses minimal computational overhead compared to full fine-tuning [S2]. Training takes 2-4 hours per experiment [S1, unofficial].
*   **Limitations:** The sources indicate that despite the adaptation, DINO-enhanced variants may not outperform standard NeRF baselines in terms of PSNR/SSIM on small datasets, suggesting potential misalignment or overfitting issues [S1, unofficial].

## Background
The method is described in the paper "Limitations of NERF with pre-trained Vision Features for Few-Shot 3D Reconstruction" [S1, unofficial]. It builds on the DINO-enhanced NeRF framework, which integrates pre-trained DINO vision features to enhance few-shot 3D reconstruction [Parent]. The specific use of LoRA for parameter-efficient fine-tuning of the DINO backbone is proposed to address the misalignment between frozen semantic features and NeRF's geometric objectives [S1, unofficial].

## Sources
[S1] Limitations of NERF with pre-trained Vision Features for Few-Shot 3D Reconstruction, https://arxiv.org/abs/2506.18208v1, fetched 2026-10-26 (unofficial)
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
