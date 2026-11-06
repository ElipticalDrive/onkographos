# MLP Neural Fields

Also called: Implicit Neural Representations, Coordinate-based Models

## Overview
MLP Neural Fields define a continuous scene function $F_\Theta: \mathbb{R}^d \rightarrow \mathbb{R}^k$ using a Multi-Layer Perceptron (MLP) that maps continuous coordinates to signal values such as density and color [S1]. This family solves the problem of representing complex 3D geometry and appearance as a compact, differentiable function, enabling novel view synthesis and high-resolution rendering without explicit mesh or voxel storage. It distinguishes itself from sibling methods like [Hash Grid Neural Fields](3855-hash-grid-neural-fields.md) by relying entirely on learned weights rather than sparse lookup tables, and from [Gaussian Head Avatar (GHA)](1478-gaussian-head-avatar-gha.md) by using a fully continuous implicit representation rather than explicit parametric primitives [S1].

## Shared principles
All methods in this family define a scene function $F_\Theta: \mathbb{R}^d \rightarrow \mathbb{R}^k$ using a neural network or a set of parameterized primitives. The input is a continuous coordinate vector (typically 3D spatial position $\mathbf{x}$, potentially augmented with viewing direction $\mathbf{d}$ or time $t$), and the output is a signal vector (e.g., density $\sigma$ and color $\mathbf{c}$). The network or primitives are trained by minimizing a loss between rendered observations and ground truth images. A critical shared challenge is **spectral bias**, where standard MLPs with smooth activations (like ReLU) struggle to learn high-frequency details, often necessitating specific architectural choices or activation functions [S1].

## Subtopics

### [NeRF MLP Architecture](910-nerf-mlp-architecture.md)
A deep MLP with positional encoding and skip connections that maps 5D coordinates (position + view direction) to density and view-dependent color. It is chosen for high-fidelity novel view synthesis where view-dependent effects (specularity) are critical. It differs from [SRN](1150-scene-representation-networks-srn.md) by modeling volumetric density rather than opaque surfaces, and from [SIREN](1250-siren-neural-field.md) by using ReLU and explicit frequency mapping.

### [Scene Representation Networks (SRN)](1150-scene-representation-networks-srn.md)
Represents a scene as an opaque surface using an MLP and a Recurrent Neural Network (RNN) for adaptive ray marching. It is chosen for simple, opaque geometry reconstruction. It differs from [NeRF](910-nerf-mlp-architecture.md) by assuming opacity and using recurrence for depth estimation rather than volumetric integration.

### [SIREN Neural Field](1250-siren-neural-field.md)
Uses sine activations and specific weight initialization to overcome spectral bias, allowing the representation of high-frequency details and well-behaved derivatives. It is chosen when fine detail reconstruction is required and standard MLPs fail. It differs from the plain [MLP Neural Field](1256-mlp-neural-field.md) by using sine activations instead of ReLU.

### [MLP Neural Field](1256-mlp-neural-field.md)
A baseline plain MLP with configurable activations (typically ReLU) mapping coordinates to signals. It serves as the primary comparison for demonstrating the impact of activation functions on spectral bias. It differs from [SIREN](1250-siren-neural-field.md) by lacking the specialized sine activations needed for high-frequency learning.

## Background
The concept of using neural networks to represent continuous signals emerged in the early 2000s, with early work on implicit surfaces [G]. The modern era of neural fields for volume rendering began with [Scene Representation Networks (SRN)](1150-scene-representation-networks-srn.md) by Mildenhall et al. (2018), which introduced the use of MLPs and RNNs for ray marching [S1]. This was followed by [NeRF](910-nerf-mlp-architecture.md) (Mildenhall et al., 2020), which popularized the use of positional encoding and volumetric rendering with MLPs [S1]. The limitation of spectral bias in ReLU-based MLPs led to the development of [SIREN](1250-siren-neural-field.md) by Sitzmann et al. (2020), which demonstrated the efficacy of sine activations for high-frequency representation [S2, unofficial].

## Sources
[S1] Mildenhall, B., et al. "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis." ECCV, 2020.
[S2] Sitzmann, V., et al. "Implicit Neural Representations with Periodic Activation Functions." NeurIPS, 2020. (unofficial)
