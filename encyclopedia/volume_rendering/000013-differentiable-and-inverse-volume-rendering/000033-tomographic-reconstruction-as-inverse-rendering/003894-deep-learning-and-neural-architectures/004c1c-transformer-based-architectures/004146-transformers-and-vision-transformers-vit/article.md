# Transformers and Vision Transformers (ViT)

Also called: Vision Transformer

## Overview
Transformers compute pairwise attention over input tokens to capture long-range dependencies that local convolutions miss [S1, unofficial]. In volume rendering, they are primarily used for super-resolution and denoising, where their ability to attend across the full spatial extent of a subvolume helps recover large-scale structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
The standard Vision Transformer (ViT) takes an image (or volume slice) as input and outputs a sequence of feature tokens. For multi-view fusion, it takes concatenated encoded patches from multiple views and outputs fused tokens containing cross-view context.

**Data Structures:**
1.  **Patches:** The input image of size $H \times W$ is divided into non-overlapping fixed-size patches of size $P \times P$.
2.  **Tokens:** Each patch is flattened and linearly projected to a latent dimension $d$, forming a token. A learnable class token (or position embeddings) is often added.
3.  **Positional Encoding:** Sinusoidal or learnable positional embeddings are added to tokens to retain spatial information [S1, unofficial].

**Steps:**
1.  **Patch Embedding:** Split input into patches and project to embedding space.
2.  **Positional Encoding:** Add positional embeddings to the patch embeddings.
3.  **Transformer Encoder:** Apply $L$ layers of Multi-Head Self-Attention (MSA) and Feed-Forward Networks (FFN).
    *   **Self-Attention:** Computes attention scores between all pairs of tokens.
    *   **Residual Connections & Layer Normalization:** Applied after MSA and FFN.
4.  **Output:** The final token sequence represents the processed features.

**Formulas:**
For a sequence of tokens $Z_0 = [x_{class}; x^1_p E; \dots; x^N_p E] + E_{pos}$, where $x^i_p$ are patch embeddings and $E$ is the linear projection matrix:
$$
Z_l = \text{MSA}(\text{LayerNorm}(Z_{l-1})) + Z_{l-1}
$$
$$
Z'_l = \text{FFN}(\text{LayerNorm}(Z_l)) + Z_l
$$
where $l$ is the layer index.

**Parameters:**
*   **Patch Size ($P$):** Commonly $16 \times 16$ [S2, unofficial].
*   **Embedding Dimension ($d$):** Commonly 768 (ViT-B) [S2, unofficial].
*   **Heads:** Commonly 12 [S2, unofficial].
*   **Layers:** Varies; Fusion ViT uses 16 layers [S2, unofficial].

**Edge Cases:**
*   **Input Size:** Must be divisible by patch size $P$.
*   **Memory:** Self-attention scales quadratically with the number of tokens ($O(N^2)$), limiting input resolution for large volumes.

## Subtopics

### Fusion ViT
A variant designed for multi-view fusion. It takes concatenated encoded image patches from $K$ views and performs all-to-all self-attention to fuse features, providing full context from all other views for spatial and context reasoning [S2, unofficial].
*   **Definition:** A 16-layer transformer similar to ViT-B [S2, unofficial].
*   **Formula:** $\{F_1, F_2, \dots, F_K\} = \mathcal{F}_{fuse}(\tilde{H}_1, \tilde{H}_2, \dots, \tilde{H}_K)$, where $F_i$ is the fused token of the $i$-th view, $\tilde{H}_i$ are the concatenated encoded image patches, and $\mathcal{F}_{fuse}$ is the fusion transformer [S2, unofficial].
*   **Parameters:** 16 layers, ViT-B/16 architecture (12 heads, embedding dimension 768, MLP ratio 4.0, 16x16 patch size), initialized from scratch [S2, unofficial].
*   **When to choose:** When integrating information from multiple sparse views (e.g., X-rays) into a coherent 3D representation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N^2)$ for self-attention, where $N$ is the number of tokens. This is higher than convolutions for local operations but more efficient for global context.
*   **Memory Complexity:** $O(N^2)$ for attention maps.
*   **Strengths:** Captures long-range dependencies; effective for recovering large-scale structures in volume super-resolution and denoising [S1, unofficial].
*   **Limitations:** High computational cost for high-resolution volumes; requires significant data for training from scratch (though pre-trained weights like DINO can be used) [S2, unofficial].

## Background
The Transformer was introduced to compute pairwise attention over input tokens [S1, unofficial]. The Vision Transformer (ViT) adapted this for images by treating fixed-size patches as tokens [S1, unofficial]. In volume rendering, these architectures have been applied to super-resolution and denoising tasks to leverage their global attention mechanism [S1, unofficial]. Fusion ViT was proposed for sparse-view X-ray to CT reconstruction to fuse multi-view features [S2, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume Visualization, https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S2] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
