# GaussianDiffusion

Also called: GaussianCube

## Overview
GaussianDiffusion is a method that trains a 3D diffusion model based on a GaussianCube representation to generate 3D Gaussian fields [S1]. It solves the problem of high reconstruction latency and multi-view inconsistency inherent in iterative optimization approaches by amortizing the reconstruction cost through learned priors [S1]. Unlike iterative methods that optimize per-scene parameters, this approach enables near-real-time sampling at test time and naturally promotes multi-view coherence [S1]. It is distinct from sibling methods like TriplaneGaussian by using a voxelized Gaussian representation converted via optimal transport rather than planar projections [S1].

## How it works
**Inputs and Outputs:**
- Input: Text prompts or single/multi-view images (implied by diffusion context [S1]).
- Output: A set of 3D Gaussians representing the scene geometry and appearance.

**Data Structures:**
- **GaussianCube**: A representation where a constant number of Gaussians are converted into a voxelized grid structure [S1]. The conversion is performed via optimal transport [S1].

**Steps:**
1. **Representation Conversion**: Convert a constant number of 3D Gaussians into the GaussianCube representation using optimal transport [S1].
2. **Diffusion Training**: Train a 3D diffusion model on the GaussianCube representation [S1].
3. **Noise Incorporation**: Incorporate structured noises from multiple viewpoints during the diffusion process to alleviate the Janus problem (inconsistent geometry across views) [S1].
4. **Variational Optimization**: Use variational 3DGS (3D Gaussian Splatting) during the generation or refinement process to mitigate floaters (artifacts) and improve generation results [S1].

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- The Janus problem (multi-view inconsistency) is a specific challenge addressed by structured noise [S1].
- Floaters (spurious geometry) are mitigated via variational 3DGS [S1].

## Subtopics

### [Direct 3D Generation](11810-direct-3d-generation.md)
Direct 3D Generation is a method that learns a feedforward or diffusion model to output Gaussian fields directly from text or images, without an explicit intermediate image set. It solves the problem of high reconstruction latency and multi-view inconsistency inherent in iterative optimization approaches by amortizing the reconstruction cost through learned priors. Unlike iterative methods that optimize per-scene parameters, this approach enables near-real-time sampling at test time and naturally promotes multi-view coherence.

### Gaussian Masking
Gaussian Masking is a technique used to prune Gaussians during training [S2]. It is applied at every densification step and for every 500 iterations after densification stops [S2]. It uses a Gaussian mask loss with a weight $\lambda_m = 0.05$, a threshold $\epsilon_m = 0.05$, and a learning rate of 0.5 for the Gaussian masks [S2].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: Not given in the sources.
- **Strengths**: Enables near-real-time sampling at test time; promotes multi-view coherence; alleviates the Janus problem [S1].
- **Limitations**: Not given in the sources.
- **When to use**: When low reconstruction latency and multi-view consistency are required, and iterative per-scene optimization is too slow [S1].
- **When not to use**: When high-fidelity per-scene optimization is preferred over generation speed, or when the scene does not benefit from diffusion priors [S1].

## Background
GaussianDiffusion was proposed in reference [S1] as a method to train a 3D diffusion model based on a GaussianCube representation. It addresses limitations in iterative 3D generation methods, such as high latency and multi-view inconsistency [S1]. The method incorporates structured noises from multiple viewpoints to alleviate the Janus problem and uses variational 3DGS to mitigate floaters [S1]. The GaussianCube representation is converted from a constant number of Gaussians with voxelization via optimal transport [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
