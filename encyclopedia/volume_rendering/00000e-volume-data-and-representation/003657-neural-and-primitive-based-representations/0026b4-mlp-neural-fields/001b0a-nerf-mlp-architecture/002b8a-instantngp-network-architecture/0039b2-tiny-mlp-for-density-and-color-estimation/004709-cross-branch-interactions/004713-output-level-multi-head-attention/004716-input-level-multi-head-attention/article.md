# Input-level Multi-head Attention

Also called: II. Input-level Multi-head Attention

## Overview
Input-level Multi-head Attention is a module that applies multi-head self-attention to hybrid encodings (concatenated view directions and hash encoding dimensions) to learn interactions between viewing directions, spatial positions, and neighboring rays [S1, unofficial]. It sets itself apart from its sibling [Output-level Multi-head Attention](11920-output-level-multi-head-attention.md) by operating on the input feature representations before density and color prediction, whereas the output-level variant operates on the preliminary predicted densities and colors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Hybrid encoding tensor $\gamma''(d, x)$ of size $|R| \times (N_x - 1 + N_d)$, where $|R|$ is the batch size (number of rays), $N_x$ is the hash encoding dimension, and $N_d$ is the view encoding dimension [S1, unofficial].
*   **Output:** Attended hybrid encoding $\gamma'''(d, x)$ of the same size as the input [S1, unofficial].

**Data Structures:**
The input tensor $\gamma''(d, x)$ is constructed by concatenating the view encoding $\gamma(d)$ with all but the first dimension of the hash encoding $\gamma_{1:N_x}(x)$ [S1, unofficial]. The first dimension of the hash encoding $\gamma_0(x)$ is excluded from this tensor to avoid over-parameterizing the view-independent density branch [S1, unofficial].

**Parameters:**
*   Number of attention heads ($H$): 2 for the LLFF dataset, 1 for the NeRF-Synthetic dataset [S1, unofficial].
*   $W_i^Q, W_i^K, W_i^V$: Learnable projection matrices for the $i$-th head [S1, unofficial].
*   $W_O$: Learnable output projection matrix [S1, unofficial].

**Steps:**
1.  Initialize Query ($Q$), Key ($K$), and Value ($V$) matrices to the input hybrid encoding $\gamma''(d, x)$ [S1, unofficial].
2.  For each head $i$ from $0$ to $H-1$:
    a. Compute projected queries, keys, and values: $Q_i = Q W_i^Q$, $K_i = K W_i^K$, $V_i = V W_i^V$ [S1, unofficial].
    b. Compute attention output: $head_i = \text{Attn}(Q_i, K_i, V_i)$ [S1, unofficial].
3.  Concatenate all head outputs: $[head_0, ..., head_H]$ [S1, unofficial].
4.  Project the concatenated result by $W_O$ to produce the final output $\gamma'''(d, x)$ [S1, unofficial].

**Formula:**
$$
\text{MultiHead}(Q, K, V) = [head_0, ..., head_H]W_O
$$
$$
head_i = \text{Attn}(QW_i^Q, KW_i^K, VW_i^V)
$$
Where $Q, K, V$ are all set to $\gamma''(d, x)$ [S1, unofficial].

**Pitfalls:**
*   **Over-parameterization:** Do not include the first dimension of the hash encoding ($\gamma_0(x)$) in the hybrid encoding $\gamma''(d, x)$. This dimension is reserved for the density MLP, which is view-independent [S1, unofficial].
*   **Dimension Mismatch:** Ensure the output projection $W_O$ maps the concatenated heads back to the input dimension size ($N_x - 1 + N_d$) [S1, unofficial].

## Subtopics

### Line Segment-based Attention Block (LSAB)
LSAB is a variant structure used in Lineformer for X-ray 3D reconstruction [S2]. It consists of a fully connected (fc) layer, two layer normalization (LN) layers, a feed-forward network (FFN), and a Line Segment-based Multi-head Self-Attention (LS-MSA) module, with skip connections [S2]. It is designed for sampling point positions on X-ray rays [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Implemented in DWTNeRF on GeForce RTX 4070 Ti Super [S1, unofficial]. Used with 2 heads for LLFF and 1 head for NeRF-Synthetic datasets [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Learns interactions between neighboring 3D points (vertical dimension of the ray batch) and between positions and viewing directions (horizontal dimension) [S1, unofficial].
*   **Limitations:** Adds computational overhead compared to simple concatenation. The specific number of heads is dataset-dependent [S1, unofficial].

**When to use:**
Use when modeling interactions between view directions and spatial features is critical, particularly in few-shot scenarios or when using hybrid encodings [S1, unofficial].

## Background
Originated in DWTNeRF, which boosts few-shot Neural Radiance Fields via Discrete Wavelet Transform [S1, unofficial]. The method was developed to explicitly model interactions between the plenoptic function's components before final prediction [S1, unofficial].

## Sources
[S1] DWTNeRF: Boosting Few-shot Neural Radiance Fields via Discrete Wavelet Transform, https://arxiv.org/abs/2501.12637v3, fetched 2026-10-27 (unofficial)
[S2] Structure-Aware Sparse-View X-Ray 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01062, fetched 2026-10-26
