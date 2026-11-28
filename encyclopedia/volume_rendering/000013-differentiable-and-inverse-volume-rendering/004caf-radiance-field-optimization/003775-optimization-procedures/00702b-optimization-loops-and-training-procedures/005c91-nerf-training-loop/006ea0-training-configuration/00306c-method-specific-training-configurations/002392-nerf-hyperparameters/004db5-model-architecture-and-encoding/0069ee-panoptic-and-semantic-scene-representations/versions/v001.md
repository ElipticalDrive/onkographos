# Panoptic and Semantic Scene Representations

Also called: Semantic NeRFs, Panoptic Neural Fields, Category-Level Semantic NeRF

## Overview
Panoptic and Semantic Scene Representations extend standard radiance fields by predicting semantic class distributions and instance identifiers alongside density and color, enabling high-level scene understanding and object-centric manipulation [S1]. Unlike standard NeRFs that only model geometry and appearance, these methods decompose the scene into "things" (discrete objects) and "stuff" (amorphous background) or predict part-level semantics, allowing for tasks such as object deletion, duplication, and category-level generalization [S1].

## Shared principles
The core mechanism involves transforming low-dimensional input coordinates into a high-dimensional feature space using sinusoidal positional encoding, which is then processed by a fully-connected MLP to predict radiance field properties.

**Positional Encoding**
The input coordinates $\mathbf{x} \in \mathbb{R}^3$ (position) and $\mathbf{d} \in \mathbb{R}^3$ (view direction) are mapped to higher dimensions using a hierarchical positional encoding function $\gamma(\cdot)$ [S1]. For a scalar input $p$, the encoding is defined as:
$$ \gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p)) $$
where $L$ is the number of frequency bands . This function is applied independently to each of the three components of the position vector and the direction vector [S1].

**Network Architecture**
The radiance field is represented by an MLP that takes the encoded position and direction as input. The architecture typically consists of two branches:
1.  **Density Branch:** Processes the encoded position features to predict volume density $\sigma$.
2.  **Color Branch:** Processes the encoded direction features concatenated with intermediate features from the density branch to predict RGB color $\mathbf{c}$ [S1].

**Ray Sampling**
Rendering requires sampling points along camera rays. A ray is parameterized as $r(\lambda) = o_{world} + \lambda d_{world}$, where $o_{world}$ is the world-space origin and $d_{world}$ is the world-space direction [S1]. The direction is computed from camera intrinsics and extrinsics: $d_{cam} = (\frac{i - c_x}{f_x}, \frac{j - c_y}{f_y}, 1)$, $d_{world} = R d_{cam}$, and $o_{world} = t$ [S1]. Position samples $x$ are generated along the ray within a defined near/far range [S1].

## Subtopics

### [Panoptic Radiance Field](15849-panoptic-radiance-field.md)
Panoptic Radiance Field learns a 3D panoptic radiance field from posed RGB images and machine-generated 2D panoptic segmentation masks, lifting noisy 2D labels into a consistent 3D representation of color, density, semantics, and instance identifiers [15849-panoptic-radiance-field.md]. Unlike [Panoptic NeRF](11916-panoptic-nerf.md) or [DM-NeRF](11916-panoptic-nerf.md) which require manual ground truth or coarse meshes, this method requires only pre-trained 2D segmentation outputs, making it robust to in-the-wild scenes [15849-panoptic-radiance-field.md]. It sets itself apart by using a linear assignment to resolve 2D instance inconsistencies and specific gradient-stopping techniques to prevent noisy labels from corrupting geometry [15849-panoptic-radiance-field.md].

### [Panoptic Neural Fields](20291-panoptic-neural-fields.md)
Panoptic Neural Fields (PNF) is an object-aware neural scene representation that decomposes a dynamic 3D scene into a set of discrete objects ("things") and an amorphous background ("stuff"), predicting a panoptic-radiance field for color, density, instance, and category labels at any point in time [20291-panoptic-neural-fields.md]. Unlike [Panoptic Radiance Field](15849-panoptic-radiance-field.md) and [Category-Level Semantic NeRF](20302-category-level-semantic-nerf.md), which are not object-aware and cannot handle dynamic scenes, PNF explicitly models moving objects using separate MLPs per instance within dynamic bounding boxes, enabling joint optimization of 3D shape, appearance, semantics, and object poses from RGB images alone [20291-panoptic-neural-fields.md].

### [Category-Level Semantic NeRF](20302-category-level-semantic-nerf.md)
Category-Level Semantic NeRF extends the standard NeRF framework to predict part segmentation logits in addition to density and color, enabling category-level generalization for articulated objects [20302-category-level-semantic-nerf.md]. It differs from sibling methods like Panoptic Radiance Field by conditioning the neural field on image features extracted from input views, allowing the model to generalize across instances of the same category rather than optimizing for a single specific object [20302-category-level-semantic-nerf.md].

### Semantic Radiance Fields
#### definition
- [S1] A method where semantic labels are formalized as a view-invariant function mapping world coordinate $x$ to a distribution over semantic labels via pre-softmax semantic logits $s(x)$, while color $c$ additionally depends on viewing direction $d$.
#### step
- [S1] Inference: Semantic logits are transformed into multi-class probabilities through a softmax normalization layer. The semantic label is determined as the class of the maximum probability in $\hat{S}(r)$.
#### formula
- [S1] $\hat{S}(r) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) s_i$ where $s_i$ are the semantic logits for sample $i$.
- [S1] $L_{2D}^{sem} = -\frac{1}{|R|} \sum_{r \in R} \sum_{l=1}^L p_l(r) \log \hat{p}_l(r)$ where $p_l(r)$ and $\hat{p}_l(r)$ are the multi-class semantic probabilities of the ground truth and predicted map for class $l$ of ray $r$.
#### layout
- [S1] Network structure: Volume densities $\sigma$ and semantic logits $s$ are functions of $(x, y, z)$ only. Color $c$ is a function of $(x, y, z)$ and $(\theta, \phi)$.

## Background
Semantic and panoptic radiance fields emerged to address the limitation of standard NeRFs in providing only low-level geometry and radiance without higher-level scene understanding [S1]. Early work formalized semantic labels as view-invariant functions mapping world coordinates to semantic distributions, allowing for the integration of 2D semantic supervision into 3D rendering [S1]. Subsequent developments introduced panoptic representations to handle both "things" and "stuff," and dynamic models to track object motion, expanding the utility of neural radiance fields for tasks like scene editing and object manipulation [S1].

## Sources
[S1] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
