# Gaussian Splatting Deformation

Also called: Dynamic 3D Gaussian Splatting, Time-varying Gaussian Splatting

## Overview
Gaussian Splatting Deformation extends static 3D Gaussian Splatting to dynamic scenes by maintaining a static canonical set of primitives and predicting time-dependent geometric and appearance offsets . It solves the problem of reconstructing and rendering moving subjects (e.g., people, fluids) from multi-view video by decomposing the 4D representation into a canonical state and a deformation function . It sets itself apart from sibling methods like [Space-time Neural Irradiance Fields](17161-space-time-neural-irradiance-fields.md) and [Dynamic NeRF](21867-dynamic-nerf.md) by using explicit differentiable rasterization of deformed Gaussians rather than neural volume rendering, and from [Neural Scene Flow Fields (NSFF)](21905-neural-scene-flow-fields-nsff.md) by predicting offsets for discrete primitives rather than continuous scene flow fields .

## Shared principles
Methods in this family handle temporal changes by either conditioning a radiance field on time, predicting explicit deformation or scene flow fields, or deforming canonical primitives [S1, unofficial]. For Gaussian Splatting variants, a static set of 3D Gaussians $\mathcal{G}_{canon}$ is defined in a canonical coordinate system, storing base parameters: mean $\boldsymbol{\mu}_i \in \mathbb{R}^3$, rotation $\mathbf{q}_i \in \mathbb{R}^4$ (unit quaternion), scale $\mathbf{s}_i \in \mathbb{R}^3$, opacity $\sigma_i$, and spherical harmonics coefficients $\mathbf{c}_{SH,i}$ . For a given time $t$, a deformation function $D$ predicts offsets $(\delta\boldsymbol{\mu}, \delta\mathbf{q}, \delta\mathbf{s})$ based on the canonical position and time . The time-varying parameters are computed as:
$$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \delta\boldsymbol{\mu} $$
$$ \mathbf{s}_i(t) = \mathbf{s}_i + \delta\mathbf{s} $$
$$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i \otimes \delta\mathbf{q}) $$
where $\otimes$ is quaternion multiplication and normalization prevents drift [G]. The deformation network $D$ typically takes spatial embeddings of canonical positions and temporal embeddings of time $t$ as input . Rendering uses standard differentiable rasterization or volume integration with the updated primitives, camera pose, and time $t$ [G]. Optimization minimizes photometric loss $\mathcal{L} = \mathcal{L}_{rec} + \lambda \mathcal{L}_{reg}$, updating Gaussian parameters and network weights via SGD .

## Subtopics

### [Sparse-Controlled 3D Gaussian Splatting](6145-sparse-controlled-3d-gaussian-splatting.md)
Sparse-Controlled 3D Gaussian Splatting (SC-GS) is a dynamic scene representation method that utilizes a small set of user-provided or learned sparse control points to drive the deformation of dense 3D Gaussians. Unlike methods that predict deformation per-Gaussian, SC-GS propagates edits across time via a learned deformation prior, resulting in a compact and editable representation. It sets itself apart from siblings like Deformable 3DGS by decoupling the deformation network's input from individual Gaussian features, instead relying on spatial interpolation from control points. See [Sparse-Controlled 3D Gaussian Splatting](6145-sparse-controlled-3d-gaussian-splatting.md).

### [Deformable 3DGS](6866-deformable-3dgs.md)
Deformable 3DGS extends static 3D Gaussian Splatting to dynamic scenes by modeling time-dependent geometric and appearance parameters. It solves the problem of reconstructing and rendering moving subjects (e.g., people, fluids) from multi-view video by decomposing the 4D representation into a static canonical set of Gaussians and a deformation function that predicts their state at any time $t$. It distinguishes itself from sibling methods like [Sparse-Controlled 3D Gaussian Splatting](6145-sparse-controlled-3d-gaussian-splatting.md) by using a direct MLP-based deformation network on per-Gaussian features rather than sparse control points. Builds on [Gaussian Splatting Deformation](22469-gaussian-splatting-deformation.md). See [Deformable 3DGS](6866-deformable-3dgs.md).

### Drive-GS
End-to-end driving simulators [S1, unofficial].

### 4D-Driving-GS
A method that drives time-of-day and weather conditioning for closed-loop simulation [S1, unofficial].

## Background
The concept of deforming canonical primitives for dynamic rendering originated in neural radiance fields (NeRF) and was adapted to 3D Gaussian Splatting as the latter gained popularity for real-time rendering. Deformable 3DGS and Sparse-Controlled 3D Gaussian Splatting represent the primary approaches to this problem within the Gaussian Splatting ecosystem . These methods are heavily used in autonomous driving simulation, where they enable the integration of dynamic actors into static backgrounds for closed-loop testing [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
