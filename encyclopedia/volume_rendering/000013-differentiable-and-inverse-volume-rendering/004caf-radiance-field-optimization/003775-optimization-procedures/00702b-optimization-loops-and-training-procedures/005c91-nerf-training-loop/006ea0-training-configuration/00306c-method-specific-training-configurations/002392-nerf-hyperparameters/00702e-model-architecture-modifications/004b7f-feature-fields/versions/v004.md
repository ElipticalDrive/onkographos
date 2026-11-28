# Feature Fields

Also called: Generalized NeRF, Semantic Fields

## Overview
Feature fields generalize the standard NeRF formulation by predicting arbitrary per-point feature signals $\mathbf{f}$ in addition to color $\mathbf{c}$ and density $\sigma$ [S1, unofficial]. This allows the radiance field to encode semantic labels, physical properties, or latent embeddings, enabling applications such as semantic segmentation, object decomposition, and self-supervised 3D understanding [S1, S2, unofficial][S3]. Unlike sibling methods that focus on multi-space representations [MS-NeRF](11707-multi-space-neural-radiance-field-ms-nerf.md) or specific geometric constraints [NRFF](12072-nrff.md), feature fields leverage the high-dimensional feature space to decouple geometry from appearance or to encode semantic/functional information [NeRF for Object Segmentation](12404-nerf-for-object-segmentation.md).

## Shared principles
The core mechanism modifies the radiance field function $F_\Theta$ to map input coordinates $(\mathbf{x}, \mathbf{d})$ to a tuple $(\mathbf{f}, \mathbf{c}, \sigma)$, where $\mathbf{f} \in \mathbb{R}^d$ is a feature vector [S1, S2, S3, S4]. The rendering of these features follows the same volume integration principle as color. For a ray $r$, the rendered feature $\hat{\mathbf{F}}(r)$ is computed as:

$$
\hat{\mathbf{F}}(r) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{f}(x_i)
$$

where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance and $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity weight [S1, S2, unofficial][S3]. Supervision is performed by comparing the rendered feature $\hat{\mathbf{F}}(r)$ against ground-truth 2D feature maps or teacher network outputs projected onto the image plane [S1, S2, unofficial][S3]. This process lifts 2D observations into a consistent 3D representation [S1, S2, unofficial][S3].

## Subtopics

### [Multi-Space Neural Radiance Field (MS-NeRF)](11707-multi-space-neural-radiance-field-ms-nerf.md)
MS-NeRF addresses the failure of standard NeRF methods to render reflective and refractive objects, which violate multi-view consistency in Euclidean space and result in blurry or distorted images. It differs from single-space NeRF by representing the scene using a group of feature fields in parallel sub-spaces rather than a single radiance field, allowing virtual images to be placed in sub-spaces visible only from specific viewpoints. This approach automatically handles mirror-like objects without manual labeling, unlike vanilla NeRF which treats virtual images as textures on reflective surfaces.

### [NRFF](12072-nrff.md)
NRFF is a method for encoding view-dependent effects in radiance fields by replacing the standard positional encoding and roughness/IDE inputs of Ref-NeRF with a structured latent representation of specular reflections. It differs from its siblings by explicitly modeling anisotropic reflections using Anisotropic Spherical Gaussians (ASG) and a custom Rendering Equation Encoding (REE) in the latent space, rather than relying on isotropic roughness or precomputed environment maps.

### [Semantic and Feature Branches](12095-semantic-and-feature-branches.md)
This family extends the standard NeRF formulation by predicting arbitrary per-point feature signals $\mathbf{f}$ in addition to color $\mathbf{c}$ and density $\sigma$, enabling applications such as semantic segmentation, gaze redirection, and pose estimation [Parent, S1, S2, S3, S4]. Unlike sibling methods that focus on multi-space representations [MS-NeRF](11707-multi-space-neural-radiance-field-ms-nerf.md) or specific geometric constraints [NRFF](12072-nrff.md), these methods leverage the high-dimensional feature space to decouple geometry from appearance or to encode semantic/functional information [NeRF for Object Segmentation](12404-nerf-for-object-segmentation.md).

### [NeRF for Object Segmentation](12404-nerf-for-object-segmentation.md)
This method extends the standard NeRF formulation to perform object segmentation by simultaneously estimating scene geometry and predicting semantic occlusion masks. It introduces "area encoding" to represent the probability that a sampling point belongs to an occluded area, leveraging the strong correlation between geometry estimation and semantic label prediction. Unlike sibling methods that predict arbitrary feature vectors, this approach specifically targets binary or probabilistic occlusion masks to enable view-consistent segmentation and subsequent inpainting of occluded regions.

### [Semantic-NeRF](14889-semantic-nerf.md)
Semantic-NeRF is a method for synthesizing semantic labels for novel views by extending the standard NeRF formulation with an additional direction-independent MLP branch. It solves the problem of 3D semantic understanding by integrating RGB images and corresponding 2D semantic labels to train the network, allowing for the recovery of semantic information from sparse or noisy annotations. It is distinguished from its siblings by being the first work to integrate semantic information directly into the NeRF framework using a dedicated semantic output branch.

### Object Field Representation
A function $f(\mathbf{p})$ that maps a 3D point $\mathbf{p}$ to an object code vector $\mathbf{o}$, representing object ownership independent of viewing angles [S2, unofficial]. The object code $\mathbf{o}$ is defined as $H+1$ dimensional, where $H$ is a predefined maximum number of solid objects the network predicts [S2, unofficial]. The last dimension is reserved for empty space to handle occlusion and collision [S2, unofficial]. The function $f$ is parameterized by a series of MLPs [S2, unofficial]. A pitfall is the variable number of objects and large non-occupied volumes, solved by fixing the dimension to $H+1$ [S2, unofficial].

### N3F
A method that learns additional feature fields to realize self-supervised 3D segmentation [S3].

### LERF
A method that learns additional feature fields to realize self-supervised 3D segmentation [S3].

## Background
The concept of feature fields generalizes the NeRF formulation where the network predicts arbitrary per-point feature signals rather than only color [S1, unofficial]. Semantic-NeRF was notably the first to introduce semantic fields within the NeRF framework, using ground-truth semantic segmentation maps to augment the radiance field with per-point semantic logits [S1, unofficial]. Subsequent works like N3F, LERF, and ISRF extended this to self-supervised 3D segmentation using teacher networks like DINO [S3]. The approach enables embodied agents to operate with spatially grounded semantics, bridging the gap between perception and reasoning [S1, unofficial].

## Sources
[S1] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
[S2] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
[S3] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
