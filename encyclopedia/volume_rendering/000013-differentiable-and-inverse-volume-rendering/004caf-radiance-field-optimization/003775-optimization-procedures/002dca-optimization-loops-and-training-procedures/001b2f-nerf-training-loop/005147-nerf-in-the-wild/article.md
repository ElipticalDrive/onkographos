# NeRF in the Wild

Also called: Martin-Brualla et al, NeRF-W

## Overview
NeRF in the Wild (NeRF-W) extends standard neural radiance fields to handle unconstrained photo collections by modeling per-image appearance variations and transient objects [S1, unofficial][S2]. Unlike standard NeRF which assumes static scenes and constant lighting, NeRF-W decomposes the scene into a static radiance field and image-specific latent codes to separate transient elements from the static background [S2].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $N$ images $\{I_i\}_{i=1}^N$ with corresponding camera poses $\{P_i\}_{i=1}^N$.
- **Outputs:** A static radiance field representation and per-image latent codes for appearance and transients.

**Data Structures:**
The method augments the standard NeRF MLP with two additional embedding tables:
1.  **Appearance Embedding ($E_{app}$):** A learnable vector $z_i \in \mathbb{R}^{d_{app}}$ for each image $i$.
2.  **Transient Embedding ($E_{trans}$):** A learnable vector $z_i \in \mathbb{R}^{d_{trans}}$ for each image $i$.

**Method Steps:**
1.  **Ray Sampling:** For each image $i$, sample rays through the camera projection.
2.  **Position Encoding:** Encode 3D positions $\mathbf{x}$ and viewing directions $\mathbf{d}$ using positional encoding (as in standard NeRF).
3.  **Static Field Query:** Query the static MLP $f_{static}$ with encoded $\mathbf{x}$ and $\mathbf{d}$ to obtain base color $\mathbf{c}_{static}$ and density $\sigma_{static}$.
4.  **Appearance Modulation:** Concatenate the appearance embedding $z_i$ with the output of the static MLP (or intermediate features) to predict the final color $\mathbf{c}_i$. This allows the network to adapt to lighting changes [S3, unofficial].
5.  **Transient Modeling:** Use the transient embedding $z_i$ to predict an additional color component $\mathbf{c}_{trans}$ or modify the density/color to account for objects that appear or disappear between images (e.g., tourists, vehicles) [S3, unofficial].
6.  **Rendering:** Perform volume rendering to compute the pixel color $C_i$ for image $i$ by integrating the static and transient contributions along the ray.
7.  **Optimization:** Minimize the reconstruction loss (e.g., MSE) between rendered images and input images. A distortion regularizer is applied to discourage floaters [S1, unofficial].

**Formulas:**
The final color $C_i$ for a ray is typically modeled as a combination of static and transient components, modulated by the appearance code. While exact architectural details vary, the core concept is:
$$ C_i = \mathcal{R}(f_{static}(\mathbf{x}, \mathbf{d}), z_i^{app}, z_i^{trans}) $$
Where $\mathcal{R}$ is the rendering integral. The appearance embedding $z_i^{app}$ captures lighting variations, and $z_i^{trans}$ captures geometric changes [S3, unofficial].

**Parameters:**
- Embedding dimensions $d_{app}, d_{trans}$: Not given in the sources.
- Number of images $N$: Varies by dataset.

**Edge Cases and Pitfalls:**
- NeRF-W cannot handle scenes with major appearance changes over long periods (e.g., seasonal changes) [S3, unofficial].
- Requires accurate camera poses; performance degrades with inaccurate poses [S3, unofficial].

## Subtopics

### [KNN-NeRF](8672-knn-nerf.md)
Uses KNN for spatial locality to improve generalization.

### [NeRV Training](10253-nerv-training.md)
Uses a differentiable volume renderer for robust training.

### [Robustness and Generalization](14094-robustness-and-generalization.md)
Techniques to improve NeRF-W's robustness to unconstrained data.

## Operating characteristics
- **Time Complexity:** Training time is longer than standard NeRF due to the additional embeddings and complexity of modeling variations [S1, unofficial].
- **Memory Complexity:** Increased memory usage due to the storage of per-image latent codes.
- **Strengths:** Handles unconstrained photo collections with varying lighting and transient objects [S1, unofficial][S2].
- **Limitations:** Cannot handle significant appearance changes over time (e.g., seasons) [S3, unofficial]. Performance depends on camera pose accuracy [S3, unofficial].
- **When to use:** When reconstructing scenes from internet photo collections (e.g., tourist landmarks) where lighting and transient objects vary [S2].
- **When not to use:** For static scenes with controlled lighting or scenes with major temporal appearance changes [S3, unofficial].

## Background
NeRF in the Wild (NeRF-W) was introduced by Martin-Brualla et al. in 2021 [S2]. It was developed to address the limitations of standard NeRF, which assumes static scenes and constant lighting, by applying neural rendering to unconstrained image sets of the same artifact [S2]. The method was tested on large-scale image collections of tourist landmarks such as the Brandenburg Gate in Berlin and the Trevi Fountain in Rome [S2]. Subsequent developments by Sun et al. (2022) used appearance embeddings but sought output in meshes rather than radiance fields [S2].

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] NEURAL RADIANCE FIELDS (NERF): REVIEW AND POTENTIAL APPLICATIONS TO DIGITAL CULTURAL HERITAGE, https://doi.org/10.5194/isprs-archives-xlviii-m-2-2023-453-2023, fetched 2026-10-29
[S3] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
