# Semantic and Feature Branches


## Overview
Semantic and Feature Branches extend the standard NeRF MLP to output auxiliary data—such as semantic labels, continuous feature vectors, or occlusion probabilities—alongside density and color, enabling tasks like segmentation, zero-shot recognition, and view-consistent inpainting [S1, S2, S3]. What sets this family apart from sibling architecture modifications (e.g., [Simple-NeRF](7371-simple-nerf.md) or [DoF-NeRF](9432-dof-nerf.md)) is the explicit decoupling of geometric/radiometric properties from high-level semantic or perceptual representations via dedicated network branches, allowing for differentiable supervision from 2D priors or sparse annotations [S1, S2, S3].

## Shared principles
The family shares the mechanism of augmenting the standard NeRF output vector. While the parent [Model Architecture Modifications](11313-model-architecture-modifications.md) describes auxiliary networks generally, the specific implementation for semantic/feature branches involves:

1.  **Branch Output:** The network outputs an additional vector $y_{aux}$ at each sampled point $x_k$ along a ray $r$, in addition to density $\sigma(x_k)$ and color $c(x_k, d_k)$.
2.  **Volume Rendering of Auxiliaries:** The auxiliary output is rendered into a pixel-space value $\hat{Y}(r)$ using the same transmittance weights $\alpha_i$ as the color channel:
    $$
    \hat{Y}(r) = \sum_{i=1}^{N} T_i \alpha_i y_{aux}(x_i)
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance [S1, S2, S3].
3.  **Supervision:** The rendered auxiliary value $\hat{Y}(r)$ is compared against a target (semantic label, teacher feature, or mask probability) using a loss function (e.g., cross-entropy, MSE) [S1, S2, S3].
4.  **Geometry Decoupling (Optional):** In feature distillation, a stop-gradient is applied to density $\sigma$ during the rendering of features to prevent inconsistent 2D teacher supervision from degrading the reconstructed geometry [S2].

## Subtopics

### [PI-NeRF](12124-pi-nerf.md)
A framework integrating NeRF with invertible neural networks (INN) to directly output the 6-DoF pose of a given image without pose initialization or iterative optimization, solving the inverse problem of view synthesis.

### [NeRF2Points](12125-nerf2points.md)
A method for generating large-scale point clouds from street view radiance fields by separating road and street scene modeling and merging them via rigid transformation.

### [GazeNeRF](12127-gazenerf.md)
A 3D-aware gaze redirection method that reformulates the task as a neural volume rendering problem. It uses a two-stream MLP architecture to separately predict volumetric features for the face and eye regions, allowing for rigid 3D rotation of the eye features to achieve gaze redirection while preserving identity.

### [HeadNeRF](12332-headnerf.md)
HeadNeRF is a NeRF-based parametric head model that integrates neural radiance fields into the parametric representation of the human head to render high-fidelity head images in real-time while supporting direct control over rendering pose and semantic attributes (identity, expression, albedo, illumination).

### Semantic-NeRF
**Definition:** A NeRF model capable of synthesizing semantic labels for novel views using an additional direction-independent MLP branch [S1].
**Step:** The branch takes position and density MLP features as input and produces a point-wise semantic label $s$. The semantic labels are generated via volume rendering:
$$
S(r) = \sum_{i=1}^{N} T_i \alpha_i s_i
$$
The semantic labels are supervised using categorical cross-entropy loss [S1]. The method supports training with sparse semantic label data (10% labeled), recovers labels from pixel-wise noise, achieves label super-resolution, and enables multi-view semantic fusion [S1].

### Distilled Feature Field (DFF)
**Definition:** A method that extends NeRF by adding a feature branch outputting a 3D feature vector $f(x)$ to describe semantics, supervised by distilling a 2D teacher network (e.g., LSeg, DINO) via volume rendering [S2].
**Step:** For a 3D coordinate $x$, the network outputs density $\sigma(x)$, color $c(x, d)$, and feature vector $f(x)$ [S2]. Volume rendering of the feature field is performed as:
$$
\hat{F}(r) = \sum_{k=1}^{K} \hat{T}(t_k) \alpha(\sigma(x_k)\delta_k) f(x_k)
$$
A stop-gradient is applied to density $\sigma$ in the rendering of features $\hat{F}(r)$ to prevent inconsistent supervision from harming geometry reconstruction [S2]. The model minimizes the sum of photometric loss $L_p$ and feature loss $L_f$:
$$
L = L_p + \lambda L_f, \quad L_p = \sum_{r \in R} \|\hat{C}(r) - C(r)\|^2, \quad L_f = \sum_{r \in R} \|\hat{F}(r) - f_{img}(I, r)\|^2
$$
where $R$ are sampled rays, $C(r)$ is ground truth color, and $\lambda$ is the weight of the feature loss [S2].
**Parameter:** Teacher network: Pretrained pixel-level image encoder, specifically `clip-ViT-B-32-multilingual-v1` from Hugging Face [S2]. $\lambda$: Weight of the feature loss $L_f$. Typical value: 0.04 [S2].
**Layout:** The model uses two networks for volume rendering with coarse-and-fine hierarchical sampling [S2].
**Comparison:** Unlike SemanticNeRF which outputs closed-set label probabilities, DFF outputs a continuous feature vector $f(x)$ to enable zero-shot segmentation with open-set text labels [S2].

### Neural Perceptual Fields (NePeRF)
**Parameter:** $\lambda_f$: weight for the feature loss term, balancing it against the color loss. Typical value: 0.04 [S2].

### Object Segmentation Component
**Definition:** A component that generates view-consistent object masks for all views from a single user-annotated mask, using depth information and a region encoding concept [S3]. The network introduces 'area encoding' for each sampling point in the NeRF model, representing the probability that the sampling point belongs to the occluded area [S3].
**Step:** Constrain the generation of correct area encoding by correctly modeling the geometry of the scene using the depth prior from the NeRF component [S3]. Obtain semantic information of whether sampling points belong to occluded areas under the guidance of the given mask [S3]. Generate view-consistent segmentation results (masks $M$) for all views [S3]. The predicted mask value $\hat{P}(r)$ for a ray $r$ is calculated by integrating the area encoding $p_i$ of $N$ sample points:
$$
\hat{P}(r) = \sum_{i=1}^{N} \left( T_i (1 - \exp(-\sigma_i \delta_i)) p_i \right), \quad \text{where } T_i = \exp \left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right)
$$
[ S3 ].

## Background
Semantic-NeRF (March 2021) was one of the early NeRF models capable of synthesizing semantic labels for novel views, built upon previous NeRF-based approaches for semantic view synthesis and scene understanding [S1]. It demonstrated that NeRF could handle sparse semantic label data and perform multi-view semantic fusion [S1]. Distilled Feature Field (DFF) and Neural Perceptual Fields (NePeRF) extended this by distilling 2D foundation models (like CLIP, LSeg, DINO) into 3D feature fields, enabling zero-shot segmentation and open-set recognition [S2]. The Object Segmentation Component framework addressed occlusion by leveraging the correlation between scene geometry estimation and semantic label prediction, using area encoding to propagate single-view masks to multiple views [S3].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S3] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
