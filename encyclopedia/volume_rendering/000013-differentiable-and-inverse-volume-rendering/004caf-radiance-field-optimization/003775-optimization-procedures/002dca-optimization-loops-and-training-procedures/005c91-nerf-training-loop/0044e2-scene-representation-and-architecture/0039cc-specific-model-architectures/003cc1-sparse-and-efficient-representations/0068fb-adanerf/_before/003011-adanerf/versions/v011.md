# AdaNeRF

Also called: AdaNeRF: Adaptive Sampling for Real-time Rendering of Neural Radiance Fields

## Overview
AdaNeRF is a method that employs fixed sample positions along each ray and incrementally introduces sparsity during training to achieve real-time rendering [S1]. It differs from sibling methods like [SinNeRF](6199-sinnerf.md) or [NeRF-RPN](8663-nerf-rpn.md) by decoupling the sampling strategy from the shading network, using a lightweight network to predict weights for all potential points on a line in one pass, thereby reducing the number of points processed by the heavier shading network [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera parameters (position, orientation, intrinsics), scene geometry (implicitly defined by the networks), and a target number of samples $N$.
*   **Output:** Rendered image pixels with color and opacity.

**Data Structures:**
The method splits the neural representation into two networks:
1.  **Sampling Network:** A lightweight MLP.
2.  **Shading Network:** A heavier MLP that processes only significant samples.

**Steps:**
1.  **Initialization:** Train a standard NeRF model to convergence to initialize the color/shading network [S1].
2.  **Architecture Split:** Replace or augment the standard sampling with a Sampling Network. This network takes ray parameters as input and outputs importance weights for a fixed set of sample positions along the ray [S1][S2, unofficial].
3.  **Weight Prediction:** The Sampling Network evaluates once per ray to predict weights $w_i$ for all candidate points $i$ on the line [S1].
4.  **Sparsity Introduction:** During training, sparsity is incrementally introduced. The Sampling Network learns to assign near-zero weights to empty space and low-importance regions [S1].
5.  **Selective Shading:** Only the samples with significant weights (predicted by the Sampling Network) are passed to the Shading Network to compute color and density [S1].
6.  **Fine-tuning:** The model is fine-tuned to the desired number of samples $N$ to create a compact neural representation [S1].
7.  **Rendering:** At inference, the Sampling Network predicts weights, and the Shading Network computes radiance for the top-$K$ weighted samples, enabling real-time performance [S1][S2, unofficial].

**Formulas:**
The rendering equation is adapted to use predicted weights $w_i$ from the Sampling Network instead of standard volume integration weights:
$$ C(r) = \sum_{i=1}^{N} w_i \cdot c_i $$
Where $w_i$ is the weight predicted by the Sampling Network for sample $i$, and $c_i$ is the color computed by the Shading Network for sample $i$. The weights $w_i$ effectively determine the contribution of each sample, allowing the method to skip empty space [S1][S2, unofficial].

**Parameters:**
*   $N$: Total number of fixed sample positions along the ray.
*   $K$: Number of significant samples passed to the Shading Network (determined by thresholding $w_i$).
*   Typical values are not explicitly given in the sources, but the method is designed to be adaptable to meet performance targets for real-time applications [S1].

**Edge Cases and Pitfalls:**
*   **Initialization:** TermiNeRF (a related approach) requires a pre-trained NeRF to initialize the color network to achieve best results; AdaNeRF follows a similar logic of splitting networks [S1].
*   **Geometric Ambiguity:** The method operates without the need for a depth map, which improves quality in geometrically ambiguous regions compared to depth-supervised methods like [ENeRF](9112-nerdi.md) [S1].

## Operating characteristics
*   **Time Complexity:** Reduced compared to standard NeRF because the heavy Shading Network is only evaluated for a subset of samples determined by the lightweight Sampling Network [S1][S2, unofficial].
*   **Memory Complexity:** Compact neural representation after fine-tuning [S1].
*   **Performance:** Capable of real-time rendering [S1].
*   **Strengths:** Adaptable to performance targets; no depth map required; reduces sampling points [S1][S2, unofficial].
*   **Limitations:** Requires a two-stage training process (initial NeRF training followed by split-network training) [S1].

## Background
AdaNeRF was introduced by Kurz et al. [KNL* 22] in the paper "AdaNeRF: Adaptive Sampling for Real-time Rendering of Neural Radiance Fields" (ECCV 2022) [S1][S2, unofficial]. It is part of the broader effort to accelerate neural radiance fields by reducing the number of sample points, alongside methods like NSVF and ENeRF [S2, unofficial].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
