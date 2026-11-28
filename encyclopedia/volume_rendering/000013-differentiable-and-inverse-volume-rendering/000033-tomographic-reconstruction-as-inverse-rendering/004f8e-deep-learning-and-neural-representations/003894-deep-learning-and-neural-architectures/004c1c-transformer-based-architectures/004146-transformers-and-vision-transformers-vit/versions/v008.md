# Transformers and Vision Transformers (ViT)

Also called: Vision Transformer

## Overview
Transformers compute pairwise attention over input tokens to capture long-range dependencies that convolutions miss [S1, unofficial]. The Vision Transformer (ViT) adapts this architecture for images by treating fixed-size patches as tokens [S1, unofficial]. Builds on [Transformer-based Architectures](13148-transformer-based-architectures.md) [G]. In volume rendering, ViT is used for super-resolution and denoising, where its ability to attend across the full spatial extent of a subvolume helps recover large-scale structures that local convolutions miss [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of tokens derived from fixed-size patches of an image or volume slice [S1, unofficial].
*   **Output:** A sequence of transformed tokens used for volume super-resolution and denoising [S1, unofficial].

**Data Structures:**
*   **Tokens:** A matrix $X \in \mathbb{R}^{N \times d}$, where $N$ is the number of patches and $d$ is the embedding dimension [G].
*   **Task Relevance Score:** A scalar or vector $TR$ used to modulate attention weights per task [S2].

**Steps:**
1.  **Patch Tokenization:** The input is divided into non-overlapping fixed-size patches, which are treated as input tokens [S1, unofficial].
2.  **Pairwise Self-Attention:** The Transformer computes attention over all pairs of input tokens to capture long-range dependencies [S1, unofficial].
3.  **Task Dynamic Modulation:** In multi-task environments, a modulation layer adjusts self-attention weights by computing a task relevance score $TR$ for each feature, optimizing weight distribution per attention head based on task-specific contextual information [S2]. Positional encodings are dynamically adjusted using the task relevance score:
    $$PE(pos, 2i) = \sin\left(pos / 10000^{2i/d_{model}}\right) \cdot TR$$
    $$PE(pos, 2i + 1) = \cos\left(pos / 10000^{2i/d_{model}}\right) \cdot TR$$
    where $pos$ is the position index, $i$ is the dimension index, $d_{model}$ is the model dimension, and $TR$ is the task relevance score [S2].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Positional Information:** The permutation-invariant nature of attention requires explicit positional encoding to preserve spatial structure [G].
*   **Computational Cost:** Self-attention scales quadratically with the number of tokens, which can be prohibitive for high-resolution volume data without optimization [G].

## Subtopics

### [Fusion ViT](13080-fusion-vit.md)
Fusion ViT is a neural module designed to fuse multi-view features by performing all-to-all self-attention over encoded image patches or rendered feature vectors from multiple input views. It solves the problem of multi-view inconsistency and incomplete context in neural rendering and tomographic reconstruction by providing each view's features with full spatial and semantic context from all other views. It distinguishes itself from standard fusion methods by leveraging the Transformer's ability to model long-range dependencies across views, either at the patch level for global reconstruction or at the pixel level for lightweight synthesis.

### Task Dynamic Modulation Layer
A layer that adjusts self-attention weights according to task demands by computing a task relevance score for each feature, allowing the model to focus on information relevant to specific tasks (texture, shape, color, depth) in a multi-task learning environment [S2]. It optimizes the weight distribution of each head in the attention mechanism by analyzing task-related contextual information [S2].

## Operating characteristics
*   **Strengths:** Captures long-range dependencies across the full spatial extent of a subvolume, effectively recovering large-scale structures that local convolutions miss [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** Volume super-resolution and denoising tasks where global context is required to recover large-scale structures [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
The Transformer architecture computes pairwise attention over input tokens to capture long-range dependencies [S1, unofficial]. The Vision Transformer (ViT) adapts this for images by treating fixed-size patches as tokens [S1, unofficial]. In the context of scientific volume visualization and rendering, Transformers have been applied to super-resolution and denoising tasks to leverage their global receptive field [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S2] 3D reconstruction of toys based on adaptive scaled neural radiation field., https://doi.org/10.7717/peerj-cs.3053, fetched 2026-10-26
