# Human and Specific Subject Modeling

Also called: Human NeRF, Dynamic Human NeRF, Volumetric Avatar

## Overview
This family modifies the NeRF architecture to represent dynamic, articulated human subjects, addressing the high-frequency geometry, non-rigid deformation, and pose-dependent appearance that static scene NeRFs cannot capture [S1][S2, unofficial]. Unlike general scene modeling, these methods integrate parametric body priors (e.g., SMPL, MANO) or semantic latent codes to disentangle rigid skeletal motion from non-rigid clothing deformation and identity-specific appearance [S1][S2, unofficial]. They distinguish themselves by conditioning the radiance field on pose, shape, and semantic information, enabling novel view synthesis for unseen poses and generalizing across subjects without per-scene optimization [S1][S2, unofficial].

## Shared principles
The baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) defines a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent color [G]. The rendering process integrates these outputs along rays using standard volume rendering equations [G].

Most modifications in this family retain the core volume rendering equation but alter one or more of the following components:
1.  **Input Space:** Augmenting $\mathbf{x}$ and $\mathbf{d}$ with additional signals such as latent codes $\mathbf{z}$, time $t$, illumination embeddings $\mathbf{z}_l$, or external feature vectors $\mathbf{f}$ from vision models [S1][S2, unofficial].
2.  **Network Topology:** Replacing the monolithic MLP with specialized branches (e.g., separate density and color networks), modular experts, or hybrid structures combining MLPs with convolutional or transformer backbones [S1][S2, unofficial].
3.  **Output Space:** Extending the output tuple $(\sigma, \mathbf{c})$ to include additional fields such as surface normals $\mathbf{n}$, semantic logits $\mathbf{s}$, uncertainty variance $\nu$, or blend weights $\Delta w$ [S1][S2, unofficial].
4.  **Representation Basis:** Substituting the coordinate-based implicit function with explicit structures like voxel grids, hash tables, Gaussian primitives, or planar surfaces, which change how features are stored and interpolated [S1][S2, unofficial].

## Subtopics

### LISA
LISA is a part-based implicit neural representation method specifically designed for modeling human hands [S1][S2, unofficial]. It addresses the complexity of hand geometry by approximating the hand as a collection of rigid parts, leveraging the MANO model for structural priors [S1][S2, unofficial]. Unlike general body avatars, LISA distinguishes itself by using Signed Distance Fields (SDFs) for shape description and implicitly encoding MANO shape parameters via skeletal posture, while simultaneously learning a color field for realistic appearance [S2, unofficial].

### [Unified Volumetric Avatar (UVA)](13784-unified-volumetric-avatar-uva.md)
Unified Volumetric Avatar (UVA) is a neural volumetric representation for human avatars that enables local and independent editing of both geometry and texture while supporting novel view and pose synthesis. It distinguishes itself from methods relying on pre-defined UV mapping or two-stage distillation by anchoring separate neural fields for geometry and appearance to a deformable mesh in canonical space, allowing direct manipulation of latent codes and end-to-end training from scratch.

### HumanNeRF
HumanNeRF is a generalizable neural radiance field method for high-fidelity free-view synthesis of dynamic humans from sparse multi-view inputs, designed to generalize to unseen subjects and camera settings without per-scene training [S2, unofficial]. It distinguishes itself from prior generalizable methods by combining aggregated pixel-alignment features with a pose-embedded non-rigid deformation field and a neural appearance blending scheme, addressing the blur artifacts and geometric discontinuities common in implicit texture representations [S2, unofficial]. Unlike per-scene methods that require hours of optimization per subject, HumanNeRF provides reasonable results immediately and achieves state-of-the-art quality with only ~1 hour of fine-tuning [S2, unofficial].

### [HumanNeRF-SE](15640-humannerf-se.md)
HumanNeRF-SE is a method for synthesizing novel pose images of humans from monocular video inputs by combining explicit SMPL shape priors with an implicit NeRF representation. It solves the problem of overfitting and high computational complexity inherent in previous NeRF-based human avatars by using frozen SMPL blending weights for rigid deformation and a sparse voxel volume to filter irrelevant sampling points. Unlike sibling methods that rely on learnable neural blend weights or frame-level features, HumanNeRF-SE uses only point-level spatial-aware features to ensure pose generalization, reducing learnable parameters to less than 1% of previous methods.

### [Dynamic Semantic Human NeRF](21763-dynamic-semantic-human-nerf.md)
Dynamic Semantic Human NeRF extends the NeRF framework to model dynamic human motion by learning a consistent continuous function for geometry and appearance variations across frames. It solves the problem of rendering humans in motion by warping observations into a canonical reference frame using the imGHUM prior, allowing a single radiance field to represent articulated human bodies. It differs from static human NeRF variants by conditioning the network on semantic latent codes (shape $\beta$, pose $\theta$, and root transformation $T$) to generalize to novel poses and shapes, and by using a residual Signed Distance Field (SDF) to capture fine-grained details like clothing deformation.

#### NeuMan Network Architecture
- **Definition:** A NeRF architecture consisting of a coarse sub-model and a fine sub-model for the scene, and a single sub-model for the human NeRF model, leveraging SMPL geometry priors [S1].
- **Layout:** Architecture visualization (Fig. 10): Blue boxes are raw inputs, green boxes are positional encoded values, black boxes are hidden layers, pink boxes are final outputs. Numbers indicate feature dimensions. '+' indicates concatenation. ReLU is applied after each hidden layer [S1].
- **Comparison:** Unlike NeuralBody which overfits to training observations and produces poor rendering on the back of the subject, NeuMan generalizes better and faithfully renders the back [S1].

#### HumanNeRF (Monocular)
- **Definition:** A method showing how to train a neural radiance field for a specific actor based on monocular input data [S2, unofficial].

## Background
Origin: The application of NeRF to human modeling emerged shortly after the initial NeRF publication [G]. Early works like NeuMan [S1] and NeuralBody [S1] demonstrated the feasibility of using SMPL priors to guide human NeRF training from monocular or sparse multi-view video. Subsequent developments, such as HumanNeRF [S2, unofficial], shifted focus toward generalizable models that do not require per-subject optimization, while methods like UVA [S1] and LISA [S1][S2, unofficial] introduced more sophisticated representations for geometry and appearance decoupling. These methods are primarily used in virtual production, digital avatars, and motion capture visualization [S1][S2, unofficial].

## Sources
[S1] NeuMan: Neural Human Radiance Field from a Single Video, https://doi.org/10.1007/978-3-031-19824-3_24, fetched 2026-10-29
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
