# Hybrid and Diffusion Representations
## Overview
Hybrid and Diffusion Representations address the challenge of synthesizing 3D Gaussian Splatting (3DGS) representations from single images or text prompts without iterative optimization, thereby amortizing reconstruction costs and enabling near-real-time sampling [S1]. This family distinguishes itself from sibling methods like [Adaptive Gaussian Refinement](11448-adaptive-gaussian-refinement.md) by decoupling spatial structure generation from attribute encoding via parallel decoders or by leveraging learned diffusion priors to solve multi-view inconsistency and spurious geometry issues [S1].
## Shared principles
Not given in the sources.
## Subtopics

### [TriplaneGaussian](7558-triplanegaussian.md)
TriplaneGaussian is a feed-forward method that generates 3DGS from a single input image by decoupling the generation of spatial structure (point cloud) and attribute encoding (tri-plane) via parallel transformer-based decoders. It leverages the geometric regularity of a tri-plane to encode Gaussian attributes, improving fidelity and consistency compared to point-cloud-only approaches.

### [GaussianDiffusion](9190-gaussiandiffusion.md)
GaussianDiffusion trains a 3D diffusion model based on a GaussianCube representation to synthesize 3D Gaussian fields directly. It solves the "Janus problem" (multi-view inconsistency) and spurious geometry ("floaters") by incorporating structured noises from multiple viewpoints and using variational 3DGS principles.

### Text2Immersion
#### definition
- [S1] A scene generation method similar to LucidDreamer but with outlier removal and two-stage optimization.
#### step
- [S1] A process removes outliers in the point cloud.
- [S1] 3DGS optimization has two stages: training coarse 3DGS and refinement.

## Background
Not given in the sources.
## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
