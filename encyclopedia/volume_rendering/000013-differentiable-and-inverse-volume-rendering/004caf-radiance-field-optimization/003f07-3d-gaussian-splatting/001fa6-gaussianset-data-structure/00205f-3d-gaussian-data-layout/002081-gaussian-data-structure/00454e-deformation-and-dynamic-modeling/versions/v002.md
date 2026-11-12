# Deformation and Dynamic Modeling

Also called: Dynamic 3DGS, 4D Gaussian Splatting, Spacetime Gaussians

## Overview
Deformation and Dynamic Modeling extends static 3D Gaussian Splatting to handle non-rigid motion and temporal changes by associating time-dependent deformation fields with Gaussian primitives. It solves the problem of rendering dynamic scenes (e.g., humans, vehicles, fluid) in real-time by decomposing the scene into a static canonical representation and a time-varying deformation mechanism. It distinguishes itself from sibling methods like [Sparse-Controlled 3D Gaussian Splatting](6145-sparse-controlled-3d-gaussian-splatting.md) by its reliance on dense per-Gaussian prediction networks rather than sparse control points, and from [4D Gaussian Splatting](https://arxiv.org/abs/2404.03414) by its use of standard MLP architectures instead of HexPlane/k-Plane decoders [S1, unofficial].

## Shared principles
All methods in this family operate on a set of canonical 3D Gaussians $\mathcal{G}_{canon}$, where each Gaussian $i$ is defined by position $\boldsymbol{\mu}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_{SH,i}$. The core mechanism involves predicting deformation deltas $(\Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i)$ based on a time parameter $t \in [0, 1]$ and potentially other features. The time-varying parameters are computed as:

$$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i $$
$$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i \otimes \Delta\mathbf{q}_i) $$
$$ \mathbf{s}_i(t) = \mathbf{s}_i + \Delta\mathbf{s}_i $$

where $\otimes$ denotes quaternion multiplication [G]. The rotation update requires normalization to prevent drift [G]. The opacity and SH coefficients are typically static or interpolated, though some variants may deform them. The rendering step rasterizes the updated Gaussians $\mathcal{G}(t)$ using standard differentiable rasterization for a given camera pose and time $t$ [S1, unofficial].

## Subtopics

### [Sparse-Controlled 3D Gaussian Splatting](6145-sparse-controlled-3d-gaussian-splatting.md)
Sparse-Controlled 3D Gaussian Splatting (SC-GS) enables user-driven editing of 4D scenes by propagating sparse control point edits across time via a learned deformation prior. It distinguishes itself from sibling methods like Deformable 3DGS by replacing per-Gaussian MLP deformation networks with a compact, control-point-driven influence model, resulting in a more editable and memory-efficient representation.

### [Deformable 3DGS](6866-deformable-3dgs.md)
Deformable 3DGS renders dynamic scenes by applying a Multi-Layer Perceptron (MLP) deformation network to per-Gaussian features over time. It is distinguished from sibling methods like [Sparse-Controlled 3D Gaussian Splatting](6145-sparse-controlled-3d-gaussian-splatting.md) by using a dense MLP predictor for all Gaussians rather than sparse control points, and from [4D Gaussian Splatting](https://arxiv.org/abs/2404.03414) by its reliance on a standard MLP architecture.

### Drive-GS
End-to-end driving simulators that use dynamic Gaussian representations for autonomous vehicle training [S1, unofficial].

### 4D-Driving-GS
A method that drives time-of-day and weather conditioning for closed-loop simulation [S1, unofficial].

### D-MiSo
A motion-aware optimization method using sparse control trajectories [S1, unofficial].

## Background
The concept of dynamic Gaussian Splatting emerged as an extension of the original 3D Gaussian Splatting [S1, unofficial]. Early approaches focused on monocular video and casual capture, using priors like CoTracker, VGG-T, DUSt3R, or metric-depth priors (e.g., MoSca, Shape-of-Motion) [S1, unofficial]. Explicit flow fields were also introduced for temporal regularization (e.g., GaussianFlow) [S1, unofficial]. By 2026, the dominant pattern for autonomous driving simulation involved per-actor and per-scene 3DGS backgrounds, forming the basis for internal simulators at companies like Tesla, Waymo, and Wayve [S1, unofficial]. Feed-forward 4D large reconstruction models (e.g., L4GM) also emerged, outputting per-frame 3DGS from single-view video [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
