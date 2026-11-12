# Implicit methods (SDF-based)
Also called: Implicit representation, MLP Implicit Surface Representation, Neural Implicit Surface

## Overview
Implicit methods parameterize scene features—such as density, color, occupancy probability, or Signed Distance Field (SDF) values—as a continuous function approximated by a Multi-Layer Perceptron (MLP) network [S1]. Unlike explicit methods (voxels, point clouds, meshes) which are discrete and limited in resolution, implicit methods are continuous and accurate at arbitrary resolutions [S1].

## Shared principles
The core mechanism involves representing the 3D scene using an implicit function approximated by an MLP [S1]. The network maps 3D coordinates in object space to a scalar output representing the specific feature (e.g., SDF value or occupancy) [S1]. This continuous representation allows for high accuracy at arbitrary resolutions due to the high function-fitting capability of MLPs [S1].

## Subtopics

### [IM-NET](52-im-net.md)
Uses VAE+GAN for training, replacing traditional reconstruction with an implicit surface function decoder for single-view 3D modeling.

### [Occupancy Networks](52-occupancy-networks.md)
Represents occupancy probability via an MLP.

## Background
Implicit methods emerged as a novel alternative to conventional reconstruction methods that relied on explicit representations like grids of voxels, point clouds, or meshes [S1]. Early implementations include IM-NET (Chen et al. 2019), which utilized VAE+GAN for deep learning training [S1], and Occupancy Networks (Mescheder et al. 2019) [S1].

## Sources
[S1] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
