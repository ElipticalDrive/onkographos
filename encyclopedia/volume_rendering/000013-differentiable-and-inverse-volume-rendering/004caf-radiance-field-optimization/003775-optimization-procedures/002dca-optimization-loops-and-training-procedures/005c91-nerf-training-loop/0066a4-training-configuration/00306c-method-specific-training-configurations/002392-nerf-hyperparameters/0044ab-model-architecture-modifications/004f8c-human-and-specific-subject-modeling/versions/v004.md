# Human and Specific Subject Modeling

Also called: Human Avatar Modeling, Dynamic Human NeRF, SMPL-NeRF Hybrids

## Overview
This family of methods extends static NeRF architectures to model dynamic human subjects, enabling novel view synthesis and pose extrapolation for animatable avatars. It addresses the limitations of standard NeRF, which overfits to static scenes and lacks the structural priors necessary to generalize across different poses or unseen subjects. By integrating parametric body models (like SMPL) or deformable mesh representations with neural radiance fields, these methods decouple geometry and appearance, allowing for high-fidelity rendering of complex human dynamics and clothing details that implicit fields alone struggle to capture [S1, S2].

## Shared principles
The core task of a NeRF network architecture is to learn a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent color. The rendering process integrates these outputs along rays using standard volume rendering equations.

## Subtopics

### [Unified Volumetric Avatar (UVA)](13784-unified-volumetric-avatar-uva.md)
UVA is a neural radiance field representation for dynamic human avatars that enables local and independent editing of both geometry and texture while supporting novel view synthesis and pose rendering. It distinguishes itself from methods relying on pre-defined UV mapping or two-stage distillation by anchoring structured latent codes to a deformable canonical mesh, allowing direct manipulation of 3D space without 3D-to-2D projection ambiguities.

### [HumanNeRF](13842-humannerf.md)
HumanNeRF is a generalizable neural representation for high-fidelity free-view synthesis of dynamic humans that generalizes to unseen subjects and camera settings using sparse RGB inputs, avoiding per-scene training through aggregated pixel-alignment features and a pose-embedded non-rigid deformation field. It differs from its siblings by combining image-based rendering with NeRF-based volume rendering into a novel neural blending scheme to address artifacts and lack of high-frequency details in synthesized textures, which are common in methods relying solely on implicit texture representation.

### [HumanNeRF-SE](15640-humannerf-se.md)
HumanNeRF-SE is a method for synthesizing novel pose images of humans from monocular video inputs by combining explicit SMPL shape priors with an implicit NeRF representation. It solves the problem of overfitting and high computational complexity inherent in previous deformable NeRF methods that learn blending weights from scratch or fine-tune SMPL weights. Unlike siblings such as [HumanNeRF](13842-humannerf.md) and [Unified Volumetric Avatar (UVA)](13784-unified-volumetric-avatar-uva.md), which often rely on frame-level features or learnable weight volumes prone to pose leakage and overfitting, HumanNeRF-SE uses frozen SMPL blending weights for rigid deformation and point-level spatial-aware features for non-rigid refinement. This ensures pose generalization without requiring multi-camera data.

#### LISA
- **Definition**: A method targeting the modeling of hands by approximating human hands with a collection of rigid parts [S1].
- **Step**: Query points are input into MLPs, which are used to predict geometry (via SDFs) and color [S1].

#### NeuMan Network Architecture
- **Definition**: A NeRF architecture consisting of a coarse sub-model and a fine sub-model for the scene, and a single sub-model for the human NeRF model, leveraging SMPL geometry priors [S2].
- **Layout**: Architecture visualization (Fig. 10): Blue boxes are raw inputs, green boxes are positional encoded values, black boxes are hidden layers, pink boxes are final outputs. Numbers indicate feature dimensions. '+' indicates concatenation. ReLU is applied after each hidden layer [S2].
- **Comparison**: Unlike NeuralBody which overfits to training observations and produces poor rendering on the back of the subject, NeuMan generalizes better and faithfully renders the back [S2].

## Background
The modeling of dynamic human subjects in neural radiance fields emerged as a distinct sub-area following the success of static NeRF. Early works like NeuralBody demonstrated the potential but suffered from overfitting and poor generalization to unseen poses [S2]. Subsequent research focused on animating subjects under novel poses, with notable contributions including NeuMan and TAVA [S1]. The field expanded to include face avatars with constraints based on animating expressions or face topology, building upon pioneering research in HyperNeRF and NeRFies [S1]. Impactful works in this domain include Neural Head Avatar (December 2021) [219], IMAvatar (December 2021) [220], and INSTA (November 2022) [221]. By 2022, the area saw further diversification with LISA targeting hand modeling [S1] and the emergence of diffusion-based 3D avatar generation powered by neural fields, such as DreamAvatar (April 2023) [222], DreamHuman (June 2023) [223], and AvatarVerse (August 2023) [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] NeuMan: Neural Human Radiance Field from a Single Video, https://doi.org/10.1007/978-3-031-19824-3_24, fetched 2026-10-29
