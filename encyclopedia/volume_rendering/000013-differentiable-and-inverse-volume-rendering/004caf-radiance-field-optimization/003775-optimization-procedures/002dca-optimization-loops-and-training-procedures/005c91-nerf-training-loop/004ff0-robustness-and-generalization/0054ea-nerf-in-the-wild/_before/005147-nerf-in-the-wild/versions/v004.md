# NeRF in the Wild

Also called: Martin-Brualla et al, NeRF-W

## Overview
NeRF in the Wild (NeRF-W) extends standard neural radiance fields to handle unconstrained photo collections by decomposing the scene into a static background and transient foreground elements, while accounting for appearance variations [S1, unofficial][S2]. It differs from standard NeRF by introducing per-image latent codes to model lighting changes and transient geometry, allowing it to reconstruct landmarks from internet photos where lighting and content vary significantly [S2][S3, unofficial].

## How it works
NeRF-W modifies the standard NeRF architecture by adding two per-image latent embeddings to the input of the final MLP layers.

**Inputs and Outputs:**
- **Inputs:** A set of $N$ images $\{I_i\}_{i=1}^N$ with corresponding camera poses $\{P_i\}_{i=1}^N$. Each image $i$ has an associated appearance latent code $a_i$ and a transient latent code $t_i$.
- **Outputs:** A rendered image for a target view, decomposed into a static component and a transient component.

**Data Structures:**
1.  **Static Radiance Field:** A standard NeRF MLP $\mathcal{F}_{static}$ that takes position $\mathbf{x}$, viewing direction $\mathbf{v}$, and outputs color $\mathbf{c}$ and density $\sigma$.
2.  **Appearance Embedding:** A learnable vector $a_i \in \mathbb{R}^{d_a}$ for each image $i$.
3.  **Transient Embedding:** A learnable vector $t_i \in \mathbb{R}^{d_t}$ for each image $i$.

**Method Steps:**
1.  **Initialization:** Initialize the static NeRF parameters and the latent vectors $a_i, t_i$ for all $N$ images.
2.  **Ray Sampling:** For each pixel in each training image $i$, cast a ray and sample points $\mathbf{x}_j$ along the ray.
3.  **Feature Extraction:**
    - Compute the static feature vector $\mathbf{f}_{static}$ using the static NeRF MLP at $\mathbf{x}_j$ and $\mathbf{v}$.
    - Retrieve the appearance embedding $a_i$ and transient embedding $t_i$ for the current image $i$.
4.  **Latent Fusion:** Concatenate the static features with the latent codes. The input to the final color/density prediction layers is typically:
    $$ \mathbf{z}_j = [\mathbf{f}_{static}(\mathbf{x}_j, \mathbf{v}); a_i; t_i] $$
    *Note: The exact fusion mechanism (concatenation vs. addition) is not explicitly detailed in the provided sources, but concatenation is the standard interpretation for "adding embeddings" to MLP inputs in this context [S3, unofficial].*
5.  **Prediction:**
    - Predict static color $\mathbf{c}_{static}$ and density $\sigma_{static}$ from $\mathbf{z}_j$.
    - Predict transient color $\mathbf{c}_{transient}$ and density $\sigma_{transient}$ from $\mathbf{z}_j$ (or a separate head).
    - The total color is the sum of static and transient contributions, modulated by transparency.
6.  **Rendering:** Perform volume rendering to compute the final pixel color $C_i$:
    $$ C_i = \sum_j T_j \alpha_j \mathbf{c}_j $$
    where $\mathbf{c}_j = \mathbf{c}_{static, j} + \mathbf{c}_{transient, j}$ and $\alpha_j$ is derived from $\sigma_{static, j} + \sigma_{transient, j}$.
7.  **Optimization:** Minimize the reconstruction loss (e.g., MSE) between rendered images and ground truth images. A distortion regularizer is applied to discourage floaters [S1, unofficial].

**Parameters:**
- Embedding dimensions $d_a, d_t$: Not given in the sources.
- Number of images $N$: Scales with the dataset size.

**Edge Cases and Pitfalls:**
- **Transient Overfitting:** If transient embeddings are too powerful, they may absorb static scene details, leading to artifacts [S3, unofficial].
- **Appearance Ambiguity:** Lighting changes can be confused with material changes if not properly regularized [S2].
- **Large Variations:** NeRF-W cannot handle scenes with major appearance changes (e.g., seasonal changes like snow) [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Training time is longer than standard NeRF due to the optimization of $2N$ additional latent vectors and the increased MLP input dimension [S1, unofficial][S2].
- **Memory Complexity:** Memory usage increases by $O(N \cdot (d_a + d_t))$ to store the latent codes [S2].
- **Strengths:** Handles unconstrained photo collections with varying lighting and transient objects (e.g., tourists, vehicles) [S1, unofficial][S2].
- **Limitations:** Cannot handle significant appearance changes over time (e.g., seasons) [S3, unofficial]. Performance degrades if transient elements are not properly separated from static geometry [S3, unofficial].

## Background
NeRF in the Wild was introduced by Martin-Brualla et al. in 2021 [S2]. It was developed to address the limitations of standard NeRF, which assumes static scenes and consistent lighting, by applying neural rendering to unconstrained image sets of artifacts [S2]. The method was tested on large-scale image collections of tourist landmarks such as the Brandenburg Gate in Berlin and the Trevi Fountain in Rome [S2]. Subsequent work by Sun et al. (2022) extended the appearance embedding concept but targeted mesh output rather than radiance fields [S2].

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] NEURAL RADIANCE FIELDS (NERF): REVIEW AND POTENTIAL APPLICATIONS TO DIGITAL CULTURAL HERITAGE, https://doi.org/10.5194/isprs-archives-xlviii-m-2-2023-453-2023, fetched 2026-10-29
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
