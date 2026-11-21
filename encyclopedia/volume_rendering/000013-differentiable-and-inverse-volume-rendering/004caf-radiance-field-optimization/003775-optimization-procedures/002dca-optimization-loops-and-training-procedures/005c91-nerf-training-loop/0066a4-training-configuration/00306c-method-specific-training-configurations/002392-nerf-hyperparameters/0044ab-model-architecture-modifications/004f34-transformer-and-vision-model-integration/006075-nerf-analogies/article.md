# NeRF Analogies

Also called: Example-Based Visual Attribute Transfer, NeRF Analogies: Example-Based Visual Attribute Transfer

## Overview
NeRF Analogies is a method for transferring visual attributes (appearance) from a source NeRF to a target NeRF by establishing dense semantic correspondences using Vision Transformer (ViT) features [S1]. It solves the problem of appearance transfer without requiring geometric alignment between scenes, relying instead on semantic similarity [S1]. It differs from sibling methods like [ViT-NeRF](13878-vit-nerf.md) and [VisionNeRF](13885-visionnerf.md) by focusing on cross-scene attribute transfer via semantic key-component matching rather than single-scene reconstruction or local appearance editing [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A source NeRF (representing the appearance to transfer) and a target NeRF (representing the geometry to retain). Input images for both scenes are used to extract features.
*   **Outputs:** A modified target NeRF that renders the target geometry with the source's appearance [S1].

**Data Structures:**
*   **NeRF MLP:** A fully connected Multi-Layer Perceptron (MLP) with 8 hidden layers of 256 neurons, followed by a single layer of 128 neurons, and an output layer activated by a Sigmoid function [S1].
*   **ViT Features:** Extracted using DiNO-ViT with a ViT-8B backbone [S1].
    *   **Patch Size:** 8 pixels.
    *   **Stride:** 4 pixels.
    *   **Resolution Enhancement:** To achieve pixel-granularity correspondences, DiNO is queried on vertically and horizontally translated versions of the image (four subsequent translations by one pixel in the -x and -y directions) [S1].
    *   **Feature Map Resolution:** For 400p images, the resulting feature maps have a resolution of 392x392, with 384 features per pixel [S1].
    *   **Feature Source:** The output of the deepest (11th) attention layer is used, specifically the **key-component** of the attention maps, which correlates well with semantic similarity [S1].

**Training Procedure:**
1.  **Feature Extraction:** Extract dense semantic features from input images of the source and target scenes using the configured DiNO-ViT setup [S1].
2.  **Correspondence Establishment:** Compute semantic similarity between source and target features to establish dense correspondences [S1].
3.  **Optimization:** Train the target NeRF to minimize a combined loss function [S1]:
    $$ \mathcal{L} = \mathcal{L}_c + \lambda \, \mathcal{L}_G $$
    Where:
    *   $\mathcal{L}_c$ is the color loss (standard NeRF rendering loss) [S1].
    *   $\mathcal{L}_G$ is the Difference of Gaussians (DoG) loss, used to enforce edge consistency and alleviate noisy correspondences [S1].
    *   $\lambda$ is the weight for the DoG loss [S1].
4.  **Loss Weighting Schedule:**
    *   For the first 20,000 iterations, $\lambda = 0$ [S1].
    *   After 20,000 iterations, $\lambda$ is gradually faded in, increasing up to a maximum of 50 [S1].
5.  **Convergence:** Training runs for a total of 60,000 iterations [S1].

**Parameters:**
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $1 \times 10^{-4}$ [S1].
*   **Batch Size:** 512 [S1].
*   **Total Training Time:** Less than two hours on a single GPU, including feature extraction [S1].

**Evaluation Setup:**
*   **Data Split:** 200 images sampled randomly from a sphere around the object: 100 for training, 20 for validation, 80 for testing [S1].
*   **Metrics:** Evaluated on unseen test views. The CLIP direction consistency metric uses 80 images rendered in a circular trajectory with a constant elevation of 30° [S1].
*   **Baseline Comparison:** Compared against NeRFStudio Instant-NGP models trained for 10,000 iterations [S1].

**Edge Cases and Pitfalls:**
*   **Noisy Correspondences:** DiNO features can produce noisy mappings where two points on the target mesh map to different points in the source NeRF. This is mitigated by using the L1 loss component of the color loss to encourage sparsity [S1].
*   **Feature Granularity:** Using DiNO-v2 is avoided because its larger patch size results in coarser feature granularity, which is unsuitable for dense pixel-granularity correspondences [S1].
*   **Alternative Features:** SIFT features perform poorly as they do not capture semantic similarities [S1]. Diffusion (hyper-) features double computation time without significantly improving correspondence quality [S1].

## Operating characteristics
*   **Time Complexity:** Feature extraction and training can be completed in less than two hours on a single GPU [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Enables appearance transfer between semantically related objects without geometric alignment; robust semantic correspondence using ViT key-components [S1].
*   **Limitations:** Performance degrades on semantically unrelated objects, though plausible outcomes may still occur if shapes are similar [S1].
*   **When to use:** When transferring appearance between scenes with similar semantic content but different geometry or lighting [S1].
*   **When not to use:** When high-frequency detail preservation is critical without semantic guidance, or when computational resources are extremely limited (though faster than diffusion-based alternatives) [S1].

## Background
NeRF Analogies was introduced by Michael Fischer, Zhengqin Li, Thu Nguyen-Phuoc, Aljaž Božič, Zhao Dong, Carl Marshall, and Tobias Ritschel from University College London and Meta Reality Labs [S1]. The method leverages the semantic understanding capabilities of Vision Transformers (specifically DiNO) to bridge the gap between different 3D representations. It builds on the principles of [Transformer and Vision Model Integration](13911-transformer-and-vision-model-integration.md) by utilizing ViT features for semantic correspondence [S1].

## Sources
[S1] NeRF Analogies: Example-Based Visual Attribute Transfer for NeRFs, https://doi.org/10.1109/cvpr52733.2024.00444, fetched 2026-10-26
