# Dynamic and Deformable Fields

Also called: Deformable Radiance Fields, Dynamic Neural Radiance Fields

## Overview
Dynamic and Deformable Fields extend static radiance field representations to model scenes that change over time, either through rigid camera motion, non-rigid object deformation, or temporal dynamics [S1][S2, unofficial]. Unlike static methods that assume a fixed geometry, these methods introduce a time dimension $t$ and a mechanism to map observed coordinates to a canonical space, enabling the synthesis of novel views for dynamic content [S1][S2, unofficial]. They are distinguished from sibling architectures like [AdaNeRF](6195-adanerf.md) or [SinNeRF](6199-sinnerf.md) by their explicit handling of temporal deformation fields or displacement vectors, rather than relying solely on static appearance embeddings or sinusoidal basis functions for time [S1][S2, unofficial][S3].

## Shared principles
All methods in this family share the mechanism of mapping 6D input coordinates (3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$) to 4D output values (density $\sigma$ and color $\mathbf{c}$) using a differentiable neural network. This allows the radiance field to be optimized through the volumetric rendering integral.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [S1][S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1][S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [S1][S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1][S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1][S3].

## Subtopics

### [Neural Deformable Voxel Grid (NDVG)](11353-neural-deformable-voxel-grid-ndvg.md)
[Neural Deformable Voxel Grid (NDVG)](11353-neural-deformable-voxel-grid-ndvg.md) is a fast deformable radiance field method for dynamic scene view synthesis that integrates voxel-grid optimization with a deformable radiance field. It replaces heavy Multi-Layer Perceptron (MLP) computations with fast voxel-grid feature interpolation to enable space-time view synthesis for dynamic scenes captured by a monocular camera. Unlike static scene applications where occlusion is determined solely by viewing direction, NDVG explicitly models occlusion in dynamic scenes, which is determined by both view direction and motion (time). It achieves comparable performance to D-NeRF but trains more than 70× faster.

### Deformable Radiance Fields
Represents a dynamic surgical scene as a static canonical radiance field and a time-dependent displacement field to handle non-rigid deformations [S1]. Preprocessing maps input coordinates and time into Fourier features using positional encoding $\gamma(\cdot)$ before feeding to the networks [S1]. Color and occupancy at time $t$ for point $\mathbf{x}$ are retrieved as $F_\Theta(\mathbf{x} + G_\Phi(\mathbf{x}, t), \mathbf{d})$ [S1]. The canonical field $F_\Theta(\mathbf{x}, \mathbf{d})$ is an 8-layer MLP with parameters $\Theta$, taking coordinates $\mathbf{x} \in \mathbb{R}^3$ and unit view direction $\mathbf{d} \in \mathbb{R}^3$ to output RGB color $\mathbf{c}(\mathbf{x}, \mathbf{d}) \in \mathbb{R}^3$ and space occupancy $\sigma(\mathbf{x}) \in \mathbb{R}$ [S1]. The displacement field $G_\Phi(\mathbf{x}, t)$ is an 8-layer MLP with parameters $\Phi$, taking space-time coordinates $(\mathbf{x}, t)$ to output the displacement vector between point $\mathbf{x}$ at time $t$ and the corresponding point in the canonical field [S1].

### Dynamic Neural Radiance Fields (TöRF)
A dynamic neural radiance field approach using two neural networks (static and dynamic) to model measurements from a moving color camera and C-ToF camera pair, enabling high-fidelity novel-view synthesis of dynamic scenes [S2, unofficial]. It blends outputs of static and dynamic networks using a position- and time-dependent blending weight $b(\mathbf{x}_t, \tau)$ predicted by the dynamic network to produce blended density $\sigma^\text{blend}$, radiance $L^\text{blend}_s$, and radiant intensity $I^\text{blend}_s$ [S2, unofficial]. The RGB radiance is modeled as:
$$
L_{\text{RGB}}(\mathbf{x}, \boldsymbol{\omega}_0, \tau) = \int_{t_n}^{t_i} T^{\text{blend}}(\mathbf{x}, \mathbf{x}_t, \tau) \sigma^{\text{blend}}(\mathbf{x}_t, \tau) L_s^{\text{blend}}(\mathbf{x}_t, \boldsymbol{\omega}_0, \tau) dt
$$
and the Time-of-Flight (ToF) radiance as:
$$
L_{\text{ToF}}(\mathbf{x}, \boldsymbol{\omega}_0, \tau) = \int_{t_n}^{t_f} \frac{T^{\text{blend}}(\mathbf{x}, \mathbf{x}_t, \tau)^2}{\|\mathbf{x} - \mathbf{x}_t\|^2} \sigma^{\text{blend}}(\mathbf{x}_t, \tau) I_s^{\text{blend}}(\mathbf{x}_t, \boldsymbol{\omega}_0, \tau) W(2 \|\mathbf{x} - \mathbf{x}_t\|) dt
$$
[S2, unofficial]. The static network $F^\text{stat}_\theta$ is a 5D function of position and direction, while the dynamic network $F^\text{dyn}_\theta$ is a 6D function of position, direction, and time $\tau$ [S2, unofficial].

### Fast Per-subject Fine-tuning
A compensation strategy to address artifacts and imperfections for unseen persons by fine-tuning specific network parameters after initial generalizable training [S3]. The network is first trained on various subjects/performers and the feature blending network $MLP_B$ is frozen [S3]. When given an unseen subject, the network parameters of the deformation field $MLP_d$ and the generalizable neural radiance field $\Phi$ are optimized [S3].

## Background
The concept of representing dynamic scenes using neural radiance fields originated with the need to model non-rigid deformations, such as those found in robotic surgery or human motion. Early work represented deformable tissues as a canonical radiance field combined with a time-dependent displacement field, allowing the scene to be viewed as a deformation of a static canonical state [S1]. This approach was later extended to handle dynamic scenes captured by moving cameras and Time-of-Flight (ToF) sensors, introducing blending mechanisms between static and dynamic network outputs to handle temporal variations [S2, unofficial]. Subsequent research addressed the challenge of generalizing to unseen subjects by proposing fine-tuning strategies that optimize deformation fields and radiance fields for specific identities while keeping general features frozen [S3]. More recently, efficiency improvements such as Neural Deformable Voxel Grids (NDVG) have been introduced to replace heavy MLP computations with voxel-grid interpolation, significantly accelerating training for dynamic scenes [S1].

## Sources
[S1] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
[S2] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
[S3] HumanNeRF: Efficiently Generated Human Radiance Field from Sparse Inputs, https://doi.org/10.1109/cvpr52688.2022.00759, fetched 2026-10-26
