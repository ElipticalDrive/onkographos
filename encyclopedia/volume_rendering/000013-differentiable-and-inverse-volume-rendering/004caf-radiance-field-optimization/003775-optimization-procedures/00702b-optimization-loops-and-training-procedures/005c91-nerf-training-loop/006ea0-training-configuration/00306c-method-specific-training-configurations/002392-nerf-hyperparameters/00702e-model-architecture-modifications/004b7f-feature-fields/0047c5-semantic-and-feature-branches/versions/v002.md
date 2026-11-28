# Semantic and Feature Branches

Also called: Auxiliary Branches, Feature Fields, Semantic NeRF

## Overview
Semantic and Feature Branches modify the standard NeRF architecture by adding an auxiliary output branch that predicts a high-dimensional vector $y_{aux}$ (representing semantic labels, continuous features, or object masks) alongside the standard density $\sigma$ and color $\mathbf{c}$ [S1, S2, S3]. This auxiliary output is rendered using the same transmittance weights $\alpha_i$ as the color channel, enabling the extraction of 3D-aware semantic or feature information from novel views [S1, S2, S3]. This approach differs from sibling methods like [Simple-NeRF](7371-simple-nerf.md) or [DoF-NeRF](9432-dof-nerf.md) which focus on pose or depth, by explicitly decoupling semantic/feature representation from appearance to enable tasks such as zero-shot segmentation, pose estimation, and scene editing [S1, S2, S3, S4].

## Shared principles
The core mechanism involves extending the radiance field function $F_\Theta$ to output an auxiliary vector $y_{aux}$ in addition to density and color. For a ray $r$ with samples $x_i$, the rendered auxiliary value $\hat{Y}(r)$ is computed as:
$$
\hat{Y}(r) = \sum_{i=1}^{N} T_i \alpha_i y_{aux}(x_i)
$$
where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance and $\alpha_i = (1 - \exp(-\sigma_i \delta_i))$ is the opacity weight [S1, S2, S3]. The auxiliary vector $y_{aux}(x_i)$ is predicted by a branch of the neural network conditioned on the 3D position $x_i$ (and optionally view direction $d_i$). This rendered output is supervised by ground-truth auxiliary labels (e.g., semantic masks, teacher network features, or area encoding probabilities) using appropriate loss functions such as categorical cross-entropy or feature distance metrics [S1, S2, S3].

## Subtopics

### [PI-NeRF](12124-pi-nerf.md)
PI-NeRF uses a feature branch to output high-dimensional ray features, which are mapped to pixel features via an Invertible Neural Network Module (INM) to enable direct 6-DoF camera pose estimation without iterative optimization. It differs from siblings by solving the inverse problem of view synthesis directly, whereas methods like [HeadNeRF](12332-headnerf.md) focus on appearance rendering.

### [NeRF2Points](12125-nerf2points.md)
NeRF2Points separates the scene into road and street components using a ray gate mask, training separate branches for each to prevent "pavement collapse" and floating artifacts common in standard NeRFs. It differs from siblings by explicitly decoupling geometry of the ground plane from vertical scene elements to stabilize gradient competition.

### [GazeNeRF](12127-gazenerf.md)
GazeNeRF employs a two-stream architecture to separately predict volumetric features for the face and eye regions, allowing for rigid 3D rotation of eye features to achieve gaze redirection while preserving identity. It differs from siblings like [HeadNeRF](12332-headnerf.md) by modeling eyes as separate 3D structures rather than a single conditioned MLP.

### [HeadNeRF](12332-headnerf.md)
HeadNeRF predicts a high-dimensional feature vector volume, decoupling geometry (controlled by identity and expression) from appearance (controlled by albedo and illumination), and uses a 2D neural renderer for real-time output. It differs from siblings by replacing final color accumulation with a 2D rendering module to reduce latency.

### Semantic-NeRF
- **Definition:** A NeRF model capable of synthesizing semantic labels for novel views using an additional direction-independent MLP branch [S1].
- **Step:** Supervise semantic labels using categorical cross-entropy loss [S1].

### Distilled Feature Field (DFF)
- **Definition:** A method that extends NeRF by adding a feature branch outputting a 3D feature vector $f(x)$ to describe semantics, supervised by distilling a 2D teacher network (e.g., LSeg, DINO) via volume rendering [S2].
- **Step:** For a 3D coordinate $x$, the network outputs density $\sigma(x)$, color $c(x, d)$, and feature vector $f(x)$ [S2]. Apply stop-gradient to density $\sigma$ in the rendering of features $\hat{F}(r)$ to prevent inconsistent supervision from harming geometry reconstruction [S2].
- **Parameter:** Teacher network: Pretrained pixel-level image encoder, specifically clip-ViT-B-32-multilingual-v1 from Hugging Face [S2]. $\lambda$: Weight of the feature loss $L_f$ in the total loss function. Typical value: 0.04 [S2].
- **Layout:** The model uses two networks for volume rendering with coarse-and-fine hierarchical sampling [S2].
- **Comparison:** Unlike SemanticNeRF which outputs closed-set label probabilities, DFF outputs a continuous feature vector $f(x)$ to enable zero-shot segmentation with open-set text labels [S2].

### Neural Perceptual Fields (NePeRF)
- **Parameter:** $\lambda_f$: weight for the feature loss term, balancing it against the color loss. Typical value: 0.04 [S2].

### Object Segmentation Component
- **Definition:** A component that generates view-consistent object masks for all views from a single user-annotated mask, using depth information and a region encoding concept [S3]. The network introduces 'area encoding' for each sampling point in the NeRF model, representing the probability that the sampling point belongs to the occluded area [S3].
- **Step:** Constrain the generation of correct area encoding by correctly modeling the geometry of the scene using the depth prior from the NeRF component [S3]. Obtain semantic information of whether sampling points belong to occluded areas under the guidance of the given mask [S3]. Generate view-consistent segmentation results (masks $M$) for all views [S3].

### Feature Volume Fusion
- **Definition:** A technique to combine feature volumes from different scenes or objects into a single composition without artifacts [S4].
- **Step:** For each voxel coordinate shared across original scenes, select the feature vector with the maximum L2 norm [S4]. Use the selected feature vector to render the fused scene, achieving tight contact between objects from different scenes [S4].
- **Parameter:** The L2 norm of the feature vector correlates with the color and density information carried by each voxel [S4].
- **Usage:** Used to fuse feature volumes for objects from different datasets, such as combining LLFF vase/stand volumes with AXYZ human subject volumes [S4].

## Background
Semantic and feature branches originated with Semantic-NeRF in March 2021, which introduced an additional direction-independent MLP branch to synthesize semantic labels for novel views, supervised by categorical cross-entropy loss [S1]. This approach allowed training with sparse semantic label data (10% labeled) and achieved label super-resolution and propagation [S1]. Subsequent work, such as Distilled Feature Field (DFF), extended this by distilling 2D teacher networks (e.g., LSeg, DINO) into 3D feature fields to enable zero-shot segmentation [S2]. The concept of feature volumes for scene manipulation was further developed in Control-NeRF, which demonstrated the fusion of feature volumes from different scenes by selecting vectors with maximum L2 norms [S4]. These methods form the basis for more complex applications like pose estimation (PI-NeRF) and gaze redirection (GazeNeRF) [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S3] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
[S4] Control-NeRF: Editable Feature Volumes for Scene Rendering and Manipulation, https://arxiv.org/abs/2204.10850, fetched 2026-10-26
