# Dynamic and Deformable Fields


## Overview
This family builds on [Specific Model Architectures](8585-specific-model-architectures.md) and extends static neural radiance fields to model scenes that change over time or undergo non-rigid deformation [G]. It solves the problem of novel-view synthesis for dynamic content by introducing a temporal or deformation dimension to the scene representation [G]. Unlike static methods, these models map 6D inputs (position, direction) plus a time or deformation parameter to radiance properties, often using displacement fields or canonical space mappings to handle motion [S1][S2, unofficial].

## Shared principles
All methods in this family map 3D position $\mathbf{x}$, viewing direction $\mathbf{d}$, and a temporal or deformation parameter $t$ (or $\tau$) to density $\sigma$ and color $\mathbf{c}$ using differentiable neural networks or feature grids [S1][S2, unofficial]. Raw coordinates and time are mapped to higher-dimensional spaces using positional encoding $\gamma(\cdot)$ to mitigate spectral bias [S1]. The radiance field is optimized by minimizing the difference between rendered pixel colors and ground truth images via differentiable volumetric rendering [S1][S2, unofficial].

## Subtopics

### [Neural Deformable Voxel Grid (NDVG)](11353-neural-deformable-voxel-grid-ndvg.md)
NDVG is a fast optimization method for dynamic scene view synthesis that integrates voxel-grid optimization with a deformable radiance field. It replaces heavy MLP computations with fast voxel-grid feature interpolation to enable space-time view synthesis for dynamic scenes captured by a monocular camera. The method consists of a deformation module that maps 3D points from observation space to a canonical space, and a canonical module that models static geometry and density using explicit voxel grids. Unlike static scene applications where occlusion is determined solely by viewing direction, NDVG explicitly models occlusion in dynamic scenes, which is determined by both view direction and motion (time). Choose it when real-time or fast optimization for monocular dynamic capture is required.

### [Fast Per-subject Fine-tuning](13107-fast-per-subject-fine-tuning.md)
Fast Per-subject Fine-tuning is a compensation strategy used in generalizable human radiance field models to address artifacts and imperfections that persist for unseen subjects after initial multi-subject training. While the base model learns common representations across a dataset, sparse input views for a new individual often result in low-frequency textures and motion artifacts. This method treats the pre-trained generalizable network as an initialization state and performs rapid optimization of specific parameters on the new subject's data. It distinguishes itself from full retraining by freezing the feature blending network, focusing adaptation only on the deformation field and the radiance field core. Choose it when adapting a pre-trained generalizable model to a new subject with sparse views.

### Deformable Radiance Fields
- **Definition:** Represents a dynamic surgical scene as a static canonical radiance field and a time-dependent displacement field to handle non-rigid deformations [S1].
- **Step:** Preprocessing maps input coordinates and time into Fourier features using positional encoding $\gamma(\cdot)$ before feeding to the networks [S1].
- **Formula:** Color and occupancy at time $t$ for point $\mathbf{x}$ are retrieved as $F_\Theta(\mathbf{x} + G_\Phi(\mathbf{x}, t), \mathbf{d})$ [S1].
- **Layout:** The canonical field $F_\Theta(\mathbf{x}, \mathbf{d})$ is an 8-layer MLP with parameters $\Theta$, mapping coordinates $\mathbf{x} \in \mathbb{R}^3$ and unit view directions $\mathbf{d} \in \mathbb{R}^3$ to RGB color $\mathbf{c}(\mathbf{x}, \mathbf{d}) \in \mathbb{R}^3$ and space occupancy $\sigma(\mathbf{x}) \in \mathbb{R}$ [S1]. The displacement field $G_\Phi(\mathbf{x}, t)$ is an 8-layer MLP with parameters $\Phi$, mapping space-time coordinates $(\mathbf{x}, t)$ into the displacement vector between point $\mathbf{x}$ at time $t$ and the corresponding point in the canonical field [S1].

### Dynamic Neural Radiance Fields (TöRF)
- **Definition:** A dynamic neural radiance field approach using two neural networks (static and dynamic) to model measurements from a moving color camera and C-ToF camera pair, enabling high-fidelity novel-view synthesis of dynamic scenes [S2, unofficial].
- **Step:** Blends outputs of static and dynamic networks using a position- and time-dependent blending weight $b(\mathbf{x}_t, \tau)$ predicted by the dynamic network to produce blended density $\sigma^\text{blend}$, radiance $L_s^\text{blend}$, and radiant intensity $I_s^\text{blend}$ [S2, unofficial].
- **Formula:** The RGB radiance is modeled as $L_{\text{RGB}}(\mathbf{x}, \boldsymbol{\omega}_0, \tau) = \int_{t_n}^{t_i} T^{\text{blend}}(\mathbf{x}, \mathbf{x}_t, \tau) \sigma^{\text{blend}}(\mathbf{x}_t, \tau) L_s^{\text{blend}}(\mathbf{x}_t, \boldsymbol{\omega}_0, \tau) dt$ [S2, unofficial]. The ToF radiance is modeled as $L_{\text{ToF}}(\mathbf{x}, \boldsymbol{\omega}_0, \tau) = \int_{t_n}^{t_f} \frac{T^{\text{blend}}(\mathbf{x}, \mathbf{x}_t, \tau)^2}{\|\mathbf{x} - \mathbf{x}_t\|^2} \sigma^{\text{blend}}(\mathbf{x}_t, \tau) I_s^{\text{blend}}(\mathbf{x}_t, \boldsymbol{\omega}_0, \tau) W(2 \|\mathbf{x} - \mathbf{x}_t\|) dt$ [S2, unofficial].

## Background
The concept of representing dynamic scenes using neural radiance fields originated with the need to model non-rigid deformations, such as those found in surgical tissues or human motion [G]. Early work, such as that described in [S1], applied this to robotic surgery, representing deformable tissues as a canonical field with a time-dependent displacement field. Concurrently, other approaches like TöRF [S2, unofficial] extended these concepts to integrate Time-of-Flight (ToF) data, using separate static and dynamic networks to handle moving cameras and dynamic scenes. These methods established the foundation for current dynamic NeRF architectures by demonstrating how displacement fields and canonical spaces could effectively model temporal changes [G].

## Sources
[S1] Neural Rendering for Stereo 3D Reconstruction of Deformable Tissues in Robotic Surgery, https://doi.org/10.1007/978-3-031-16449-1_41, fetched 2026-10-26
[S2] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
