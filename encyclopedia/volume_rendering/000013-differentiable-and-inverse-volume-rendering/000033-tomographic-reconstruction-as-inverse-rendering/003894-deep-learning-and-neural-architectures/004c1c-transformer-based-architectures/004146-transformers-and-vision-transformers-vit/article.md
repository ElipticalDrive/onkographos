# Transformers and Vision Transformers (ViT)
Also called: Vision Transformer
## Overview
Transformers compute pairwise attention over input tokens to capture long-range dependencies that convolutions miss [S1, unofficial]. The Vision Transformer (ViT) adapts this architecture for images by treating fixed-size patches as tokens [S1, unofficial]. In volume rendering, these methods are used for super-resolution and denoising, where their ability to attend across the full spatial extent of a subvolume helps recover large-scale structures that local convolutions miss [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A sequence of tokens representing image patches or volume subvolume features.
- Output: Processed token embeddings with global context aggregated.

**Data Structures:**
- Tokens: A sequence of vectors $X = [x_1, x_2, ..., x_N]$, where $N$ is the number of patches/tokens.
- Embeddings: Each token $x_i \in \mathbb{R}^d$.

**Steps:**
1. **Tokenization:** The input image or volume slice is divided into fixed-size patches. Each patch is flattened and linearly projected to form the initial token embeddings [S1, unofficial].
2. **Positional Encoding:** Positional information is added to the token embeddings to preserve spatial structure, as the self-attention mechanism is permutation-invariant [S1, unofficial].
3. **Self-Attention:** For each token, compute Query ($Q$), Key ($K$), and Value ($V$) vectors via learned linear projections. Compute attention scores using the scaled dot-product:
   $$ \text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V $$
   where $d_k$ is the dimension of the key vectors [S1, unofficial]. This allows each token to attend to all other tokens in the sequence.
4. **Multi-Head Attention:** The attention mechanism is repeated in parallel across multiple heads to allow the model to jointly attend to information from different representation subspaces [S1, unofficial].
5. **Feed-Forward Network:** Each token's representation is passed through a position-wise feed-forward network (typically two linear layers with a non-linearity in between) [S1, unofficial].
6. **Residual Connections and Layer Normalization:** Additive residual connections and layer normalization are applied after the attention and feed-forward steps to stabilize training [S1, unofficial].

**Parameters:**
- Patch size: Determines the number of tokens $N$.
- Number of attention heads: Controls the parallel attention subspaces.
- Hidden dimension $d$: The size of the token embeddings.

**Edge Cases and Pitfalls:**
- **Boundary Effects:** Patches at the boundary of the volume or image may lack context from outside the frame. Padding or specific boundary handling may be required.
- **Computational Cost:** The self-attention mechanism has quadratic complexity with respect to the number of tokens, which can be prohibitive for high-resolution volume data [S1, unofficial].
- **Local Structure Loss:** While global context is captured, fine-grained local details might be smoothed out if the attention weights are too diffuse.

## Subtopics
## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N^2 \cdot d)$ for the self-attention mechanism, where $N$ is the number of tokens and $d$ is the embedding dimension.
- Memory: $O(N^2)$ to store the attention matrix.

**Measured Performance:**
- Transformers excel in recovering large-scale structures in volume super-resolution and denoising tasks compared to local convolutions [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Ability to model long-range dependencies; global context aggregation.
- **Limitations:** High computational cost for large $N$; potential loss of local detail; requires significant data for training.

**When to Use:**
- Use when global structural context is critical, such as in volume super-resolution or denoising where artifacts span large areas.
- Avoid when computational resources are limited or when local features dominate the task.

## Background
The Transformer architecture was originally introduced for sequence modeling tasks [S1, unofficial]. The Vision Transformer (ViT) adapted this for image processing by treating image patches as tokens [S1, unofficial]. In the context of scientific volume data, these methods have been applied to super-resolution and denoising to leverage their global context modeling capabilities [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
