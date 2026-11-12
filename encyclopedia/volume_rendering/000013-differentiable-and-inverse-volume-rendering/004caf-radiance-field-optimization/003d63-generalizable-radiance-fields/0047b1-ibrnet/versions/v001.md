# IBRNet

Also called: Instant Neural Graphics Primitives with Image-Based Rendering Networks

## Overview
IBRNet is a feedforward neural network for generalizable view synthesis that bypasses per-scene optimization by learning a data prior from a training set [S1]. It differs from standard NeRF by using a CNN to extract multi-view features and a Ray Transformer to predict density, enabling inference on unseen scenes without iterative training [S1].

## How it works
**Inputs:** A target view (camera pose $P_{tgt}$) and a set of $N$ source views $\{I_i, P_i\}_{i=1}^N$ selected from the training set.
**Outputs:** Volume density $\sigma$ and color $C$ along rays from the target view.

**Data Structures:**
1.  **Source Views:** $N$ images $I_i$ with known camera matrices $P_i$.
2.  **CNN Features:** Feature maps $F_i$ extracted from $I_i$.
3.  **Ray Transformer:** A Transformer encoder-decoder architecture processing ray segments.

**Steps:**
1.  **View Selection:** Select $N$ views from the training set whose viewing directions are most similar to the target view [S1].
2.  **Feature Extraction:** Pass each source image $I_i$ through a CNN to extract feature maps $F_i$ [S1].
3.  **Ray Querying:** For a query point $x$ on a ray from the target camera:
    a. Project $x$ onto each source image plane using $P_i$ to obtain 2D coordinates.
    b. Sample color $c_i$ and feature vector $f_i$ from $I_i$ and $F_i$ at these coordinates [S1].
4.  **Feature Refinement:** Pass the set of pairs $\{(c_i, f_i)\}_{i=1}^N$ through an MLP. This MLP refines features to $f'_i$ (making them multi-view aware) and produces pooling weights $w_i$ [S1].
5.  **Aggregation:** Compute the aggregated feature $F_{agg}$ for point $x$:
    $$ F_{agg}(x) = \sum_{i=1}^N w_i f'_i $$
    This aggregation is used for density prediction [S1].
6.  **Ray Processing:** Repeat steps 3-5 for all query points along the ray. Concatenate the results (features and/or density estimates) along the ray dimension.
7.  **Density Prediction:** Feed the concatenated ray representation into a Ray Transformer [S1]. The Transformer outputs the final volume density $\sigma$ (and potentially color) for the ray [S1].

**Parameters:**
*   $N$: Number of source views. Typical values are small (e.g., 3-10) [S1].
*   CNN Backbone: Not specified in sources [G].
*   MLP: Refines features and computes weights [S1].
*   Ray Transformer: Processes the aggregated ray features [S1].

**Edge Cases:**
*   **View Selection:** If no similar views exist, performance degrades [S1].
*   **Occlusion:** The method assumes visibility; occluded points may produce artifacts if not handled by the Transformer [G].
*   **Precision:** Floating-point precision in projection and sampling affects feature alignment [G].

## Subtopics

### NaN
A denoising method built on IBRNet that incorporates inter-view and spatial awareness to enhance noise robustness [S1]. It achieves state-of-the-art results in burst denoising under challenging conditions like large motion and high noise [S1]. It avoids per-scene training by leveraging IBRNet's generalization to unseen scenes with minimal input [S1].

## Operating characteristics
*   **Time Complexity:** Inference is feedforward, significantly faster than iterative NeRF optimization [S2, unofficial].
*   **Memory Complexity:** Depends on the number of source views $N$ and the size of the CNN/Transformer [S1].
*   **Performance:** Widely used in benchmarks for few-shot or sparse-view reconstruction [S1].
*   **Strengths:** Generalizes to unseen scenes; no per-scene training required; fast inference [S1].
*   **Limitations:** Requires a training set of similar scenes; performance depends on the quality of view selection [S1].
*   **When to use:** Sparse-view synthesis where per-scene optimization is too slow [S1].
*   **When not to use:** Dense-view scenarios where iterative optimization might yield higher quality [G].

## Background
IBRNet was published in February 2021 as a NeRF-adjacent method for view synthesis [S1]. It was designed to address the slow training times of NeRF by using a feedforward model to learn a data prior [S2, unofficial]. It is widely used in benchmarks for few-shot or sparse-view reconstruction [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
