# Hybrid and Diffusion Representations

Also called: Generative 3DGS, Feed-forward 3DGS, Diffusion-based 3DGS

## Overview
Hybrid and Diffusion Representations address the problem of generating 3D Gaussian Splatting (3DGS) scenes from single images or text prompts without iterative per-scene optimization. Unlike sibling methods that rely on gradient-based densification [Densification Strategy](6234-densification-strategy.md) or score-based refinement [Score-based Densification](4705-score-based-densification.md), these approaches use learned generative priors to predict a complete set of Gaussians in a single forward pass [S1]. They are distinguished by their ability to synthesize geometry and attributes directly from conditions, bypassing the need for initial point cloud initialization and subsequent gradient descent.

## Shared principles
The family shares the objective of mapping a low-dimensional condition (image or text) to a high-dimensional 3D Gaussian set $\mathcal{G} = \{ \mu_i, \Sigma_i, \alpha_i, c_i \}_{i=1}^N$. The core mechanism involves a generative model that outputs either explicit Gaussian parameters or an intermediate representation (such as a voxel grid or point cloud) that is decoded into Gaussians. A key shared challenge is the "Janus problem" (inconsistent geometry across viewpoints) and the generation of spurious "floaters," which these methods mitigate through structured noise, variational objectives, or multi-stage refinement [S1].

## Subtopics

### [TriplaneGaussian](7558-triplanegaussian.md)
TriplaneGaussian is a feed-forward method that decouples geometric primitive generation from attribute encoding. It solves the single-image reconstruction problem by generating a sparse point cloud and a tri-plane feature encoder, then fusing them via projection. It differs from diffusion-based siblings by using a transformer-based point cloud decoder and tri-plane querying rather than a voxelized diffusion process. It is chosen when high-fidelity single-image reconstruction is required without iterative optimization. See [TriplaneGaussian](7558-triplanegaussian.md).

### [GaussianDiffusion](9190-gaussiandiffusion.md)
GaussianDiffusion trains a 3D diffusion model on a **GaussianCube** representation to generate 3DGS scenes directly from conditions. It addresses the Janus problem and floaters by incorporating structured multi-view noise and variational objectives. It differs from TriplaneGaussian by operating in a voxelized latent space via optimal transport rather than explicit point-cloud-to-triplane projection. It is chosen for text-to-3D or image-to-3D generation where multi-view consistency is critical. See [GaussianDiffusion](9190-gaussiandiffusion.md).

### Text2Immersion
Text2Immersion is a scene generation method similar to LucidDreamer but incorporates outlier removal and two-stage optimization [S1]. It removes outliers in the point cloud and performs 3DGS optimization in two stages: training coarse 3DGS and refinement [S1].

## Background
The shift from iterative optimization to generative representations emerged as a solution to the high computational cost and instability of initializing 3DGS from sparse structures. Early methods relied on Structure-from-Motion (SfM) points, which were insufficient for single-image inputs. The introduction of hybrid architectures (combining transformers and tri-planes) and diffusion models on Gaussian representations allowed for direct synthesis. This family is primarily used in text-to-3D and single-image 3D reconstruction pipelines [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
