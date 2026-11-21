# Multi-stage and Progressive Training


## Overview
Multi-stage and Progressive Training refers to strategies that decouple the optimization of 3D Gaussian Splatting (3DGS) parameters or data streams to improve stability, convergence, or real-time capability. It solves the problem where joint optimization of complex attributes (e.g., materials, codecs) or continuous data ingestion leads to interference, instability, or overfitting [S1]. It sets itself apart from single-stage training by separating concerns into distinct phases or temporal streams, and from other methods by specifically targeting parameter decoupling (Two-stage) or incremental field updates (On-the-Fly) [S1].

## Shared principles
The family shares the core mechanism of temporal or parametric decoupling. In multi-stage approaches, the optimization domain is split into sequential phases where base parameters (geometry, appearance) are stabilized before refining specific attributes (materials, latents) while keeping the base fixed [S1]. In progressive approaches, the optimization domain is split temporally, allowing the Gaussian field to be updated incrementally as new data arrives, balancing the stability of existing geometry with the plasticity required for new data [S1]. Both approaches rely on iterative gradient descent with specific loss functions tailored to the active phase or data subset.

## Subtopics

### [Two-stage Optimization](14857-two-stage-optimization.md)
Decouples geometric/appearance parameters from material or compression parameters to prevent interference. Choose when optimizing complex attributes like PBR materials or entropy-coded latents that cause instability in joint training.

### [On-the-Fly GS Progressive Training](17502-on-the-fly-gs-progressive-training.md)
Optimizes 3DGS fields incrementally as new images are acquired, enabling near real-time reconstruction. Choose when processing continuous image streams or large-scale scenes where offline retraining is infeasible.

## Background
The concept of decoupling optimization stages originates from the need to handle complex attribute sets in 3DGS, such as those required for physically based rendering (GS-IR) or efficient compression (CodecSplat) [S1]. Progressive training for 3DGS emerged to address the computational cost of retraining on large datasets, enabling incremental updates via On-the-Fly SfM integration [S1].

## Sources
[S1] SpotLessSplats: Ignoring Distractors in 3D Gaussian Splatting, https://doi.org/10.1145/3727143, fetched 2026-10-27
