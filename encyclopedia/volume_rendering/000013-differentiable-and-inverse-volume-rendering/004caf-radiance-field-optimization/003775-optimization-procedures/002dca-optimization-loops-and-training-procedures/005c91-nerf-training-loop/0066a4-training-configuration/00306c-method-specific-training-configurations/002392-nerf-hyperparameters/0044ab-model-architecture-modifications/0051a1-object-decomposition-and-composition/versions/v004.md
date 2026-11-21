# Object Decomposition and Composition

Also called: Object Decomposition, Compositional NeRF, Scene Decomposition

## Overview
Object Decomposition and Composition refers to a family of NeRF architectures that explicitly separate a scene into distinct semantic or geometric entities (objects) to enable instance-level editing, manipulation, and analysis. Unlike standard NeRFs that treat the scene as a monolithic volume, these methods introduce mechanisms to identify, isolate, and recombine individual objects. This family is distinguished from siblings like [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md) by its focus on semantic/instance separation rather than just boundary representation, and from [Mixture-of-Experts and Modular Architectures](14429-mixture-of-experts-and-modular-architectures.md) by its specific goal of compositional scene editing rather than just computational efficiency.

## Shared principles
The core task of a NeRF network architecture is to learn a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c}$ is the view-dependent color. The rendering process integrates these outputs along rays using standard volume rendering equations.

## Subtopics

### [DM-NeRF](13525-dm-nerf.md)
Simultaneously reconstructs geometry, decomposes objects, and supports manipulation via a learnable object field.

### [Unified Decompositional and Compositional NeRF](14413-unified-decompositional-and-compositional-nerf.md)
Two-stage framework using a coarse global field to guide a fine-grained decomposed field, re-composed via differentiable one-hot weights.

### [Detail-based Segmentation](15935-detail-based-segmentation.md)
Divides scenes by object detail frequency, assigning high-frequency objects to individual NeRFs and low-frequency ones to a joint NeRF.

### Fig-NeRF
- [S1] Scene composition model for object interpolation and amodal segmentation using two separate NeRF models for foreground and background.
- [S1] Use an appearance NeRF conditioned on latent codes for the background model.
- [S1] Use two photometric losses, one for the foreground and one for the background.

## Background
The concept of decomposing scenes into compositional parts emerged early in NeRF research. Fig-NeRF (April 2021) [S1] focused on object interpolation and amodal segmentation using separate NeRFs for foreground and background. Object-NeRF (September 2021) [S1] introduced a hybrid voxel-based approach with separate NeRFs for objects and the scene, conditioned on interpolated voxel features. These works laid the groundwork for later methods that integrate decomposition directly into the optimization loop for editing.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
