# Multi-View Epipolar Encoder

Also called: Multi-View Epipolar Extension

## Overview
The Multi-View Epipolar Encoder extends the standard epipolar encoder from pairs of images to an arbitrary number of reference views. It solves the problem of limited geometric context in two-view reconstruction by aggregating epipolar samples from multiple input images. It differs from its parent, the [Epipolar Encoder](6426-epipolar-encoder.md), by replacing the single epipolar line's samples with the union of samples from $N$ views and introducing view-specific embeddings to disambiguate the source of each sample.

## How it works
**Inputs and Outputs:**
- Input: A target pixel coordinate $(u, v)$, $N$ reference images with known camera poses, and the corresponding epipolar geometry.
- Output: A feature tensor representing the aggregated epipolar context for the target pixel, ready for the epipolar transformer.

**Data Structures:**
- **Learnable Per-Image Embedding:** A vector $e_i \in \mathbb{R}^d$ for each reference image $i \in \{1, \dots, N\}$. These are trainable parameters shared across all pixels.
- **Sampled Features:** For each reference image $i$, a set of features $F_i$ sampled along the epipolar line corresponding to $(u, v)$.

**Steps:**
1. **Epipolar Sampling:** For each reference image $i \in \{1, \dots, N\}$, compute the epipolar line in image $i$ corresponding to the target pixel $(u, v)$ in the reference view. Sample features $F_i$ along this line using the same sampling strategy as the base [Epipolar Encoder](6426-epipolar-encoder.md).
2. **View Embedding Addition:** For each sampled feature $f \in F_i$, add the learnable per-image embedding $e_i$ to $f$. This allows the subsequent transformer to distinguish which reference image a sample originated from.
   $$ f'_i = f_i + e_i $$
3. **Union Aggregation:** Collect all modified features from all $N$ images into a single set.
   $$ \mathcal{F}_{union} = \bigcup_{i=1}^{N} F'_i $$
4. **Transformer Input:** Pass $\mathcal{F}_{union}$ as the input sequence to the epipolar transformer. The transformer processes these samples to produce the final encoded representation.

**Parameters:**
- $N$: Number of reference views. The method supports arbitrary $N$, though the source demonstrates $N=3$ [S1].
- Learning Rate: The source notes a reduction to $1 \times 10^{-4}$ from the default $3 \times 10^{-4}$ to prevent training collapse, but this is a training hyperparameter, not a method parameter [S1].

**Edge Cases and Pitfalls:**
- **Computational Cost:** The number of samples grows linearly with $N$. If the sampling density is high, memory and compute requirements increase significantly.
- **Ambiguity:** Without the learnable embeddings, the transformer cannot distinguish between samples from different views, leading to potential confusion in feature aggregation.

## Subtopics

### [Plane Sweep Volume (PSV)](19017-plane-sweep-volume-psv.md)
The Plane Sweep Volume (PSV) is a 5D tensor representation that encodes the epipolar geometry between a set of input views and a target view by projecting input images onto a series of depth planes. It differs from standard volume representations by structuring data such that adjacent camera rays have adjacent encodings, allowing local operators to process epipolar lines efficiently.

## Operating characteristics
- **Time Complexity:** Linear in the number of reference views $N$ and the number of samples per view.
- **Memory Complexity:** Linear in $N$ and the number of samples.
- **Performance:** The 3-view variant achieves higher PSNR (28.31 vs 26.09) and SSIM (0.908 vs 0.863) on the Real Estate 10k dataset compared to the 2-view baseline [S1].
- **Strengths:** Improved reconstruction quality with additional views; scalable to arbitrary numbers of views.
- **Limitations:** Increased computational cost with more views; potential for training instability if learning rate is not adjusted [S1].

## Background
The Multi-View Epipolar Encoder was introduced in the PixelSplat paper (2024) as an extension to the original two-view epipolar encoder [S1]. It was developed to leverage more reference views for scalable generalizable 3D reconstruction from images.

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
