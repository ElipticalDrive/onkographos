# Hybrid and Diffusion Representations

Also called: Generative 3D Gaussian Splatting, Gaussian Diffusion, Triplane Gaussian

## Overview
Hybrid and Diffusion Representations extend 3D Gaussian Splatting (3DGS) from a discriminative reconstruction technique to a generative framework, enabling scene synthesis from single images or text prompts without iterative per-scene optimization. Unlike standard differentiable training loops that optimize Gaussian parameters via gradient descent to fit observed views, these methods employ feed-forward networks or diffusion models to directly predict Gaussian attributes [S1]. This approach solves the latency bottleneck of inverse rendering while addressing generation-specific artifacts such as the "Janus problem" (geometric inconsistency across viewpoints) and "floaters" (spurious low-opacity primitives) [S1].

## Shared principles
The family shares a departure from the iterative densification loop described in [Densification Strategy](6234-densification-strategy.md). Instead of computing gradient norms or opacity metrics to trigger split/clone actions, these methods rely on learned priors:
1.  **Generative Prior:** A neural network (e.g., transformer, U-Net, or diffusion model) is trained to map input conditions (image features, text embeddings) to Gaussian parameters or a structured latent representation (e.g., Tri-planes, GaussianCube).
2.  **Direct Prediction:** The output is a complete set of Gaussians $\mathcal{G} = \{ \mu_i, \Sigma_i, \alpha_i, c_i \}$ predicted in a single forward pass, bypassing the need for per-scene gradient descent [S1].
3.  **Artifact Mitigation:** Specific architectural choices (structured noise, variational objectives) are employed to enforce multi-view consistency and suppress spurious geometry, which are common failure modes in naive generative 3DGS [S1].

## Subtopics

### [TriplaneGaussian](7558-triplanegaussian.md)
TriplaneGaussian is a feed-forward method for generating 3D Gaussian Splatting representations from a single input image. It solves the problem of single-image 3D reconstruction by combining a transformer-based point cloud decoder with a tri-plane feature encoding scheme. Unlike standard differentiable training loops that optimize Gaussian parameters via gradient descent, TriplaneGaussian directly predicts Gaussian attributes through a hybrid generative network, enabling fast inference without iterative optimization. It is distinguished from sibling methods like AGG by its specific architecture of projecting a densified point cloud onto tri-plane features to query and augment attributes, whereas AGG generates a coarse 3DGS first and upsamples it via a U-Net. See [TriplaneGaussian](7558-triplanegaussian.md) for implementation details.

### [GaussianDiffusion](9190-gaussiandiffusion.md)
GaussianDiffusion is a method that trains a 3D diffusion model based on a **GaussianCube** representation. It addresses the "Janus problem" (inconsistent geometry across viewpoints) and the generation of "floaters" (spurious, low-opacity primitives) common in 3D Gaussian Splatting (3DGS) generation. It sets itself apart from standard 3DGS training by incorporating structured multi-view noise into the diffusion process and using variational 3DGS principles to mitigate floaters. See [GaussianDiffusion](9190-gaussiandiffusion.md) for implementation details.

#### Text2Immersion
*   **Definition:** A scene generation method similar to LucidDreamer but with outlier removal and two-stage optimization [S1].
*   **Process:** A process removes outliers in the point cloud [S1]. 3DGS optimization has two stages: training coarse 3DGS and refinement [S1].

## Background
The integration of generative models with 3D Gaussian Splatting emerged as a solution to the high computational cost of per-scene optimization. Early works like TriplaneGaussian [26] and AGG [278] introduced hybrid architectures that predict Gaussian attributes from image features, enabling single-image 3D reconstruction [S1]. Subsequent research, such as GaussianDiffusion, applied diffusion models to structured Gaussian representations (GaussianCube) to improve geometric consistency and reduce artifacts like floaters [S1]. The field has expanded to include text-to-3D methods (e.g., Text2Immersion, GALA3D, DreamScene) that leverage diffusion priors for scene composition and generation [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
