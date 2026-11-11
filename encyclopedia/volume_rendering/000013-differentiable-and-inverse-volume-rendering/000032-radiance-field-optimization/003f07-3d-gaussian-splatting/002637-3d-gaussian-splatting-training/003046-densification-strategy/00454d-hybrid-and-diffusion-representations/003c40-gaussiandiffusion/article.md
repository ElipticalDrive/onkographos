# GaussianDiffusion

Also called: GaussianCube

## Overview
GaussianDiffusion is a method that trains a 3D diffusion model based on a **GaussianCube** representation to generate 3D Gaussian Splatting (3DGS) scenes directly from input conditions [S1]. It addresses the "Janus problem" (inconsistent geometry across viewpoints) and the generation of spurious "floaters" common in naive generative 3DGS by incorporating structured multi-view noise and variational objectives [S1]. Unlike iterative densification strategies [Densification Strategy](6234-densification-strategy.md), it relies on a learned generative prior to predict a complete set of Gaussians in a single forward pass, bypassing per-scene gradient descent [S1].

## How it works
**Inputs:** A condition vector (e.g., text embedding or image features) and a noise schedule.
**Outputs:** A set of 3D Gaussians $\mathcal{G} = \{ \mu_i, \Sigma_i, \alpha_i, c_i \}$ encoded in a GaussianCube structure.

**Data Structures:**
1.  **GaussianCube:** A voxelized representation of a constant number of Gaussians. It is constructed by converting a fixed-size set of Gaussians into a voxel grid using optimal transport [S1].
2.  **Diffusion Model:** A neural network (e.g., U-Net or Transformer) that predicts noise or denoised GaussianCube states.

**Steps:**
1.  **Preprocessing (GaussianCube Construction):**
    *   Given a target scene or training data, represent it as a fixed number of Gaussians.
    *   Apply optimal transport to map these Gaussians to a voxel grid, creating the GaussianCube representation [S1].
2.  **Training the Diffusion Model:**
    *   Add structured noise from multiple viewpoints to the GaussianCube to enforce multi-view consistency [S1].
    *   Train the model to predict the noise or the clean GaussianCube state using a variational objective [S1].
    *   The variational objective helps mitigate floaters (spurious geometry) by regularizing the generation process [S1].
3.  **Inference:**
    *   Sample from the diffusion model conditioned on the input.
    *   Decode the resulting GaussianCube back into a set of 3D Gaussians $\mathcal{G}$ [S1].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Janus Problem:** Without structured multi-view noise, generated objects may exhibit inconsistent geometry from different angles [S1].
*   **Floaters:** Naive generation often produces floating artifacts; variational 3DGS is required to suppress these [S1].
*   **Optimal Transport Complexity:** Converting Gaussians to GaussianCube via optimal transport can be computationally intensive [S1].

## Subtopics

### Gaussian Masking
A technique used to prune Gaussians during training. It applies Gaussian masks at every densification step and for every 500 iterations after densification stops [S2].
*   **Loss Weight ($\lambda_m$):** 0.05 [S2].
*   **Threshold ($\epsilon_m$):** 0.05 [S2].
*   **Learning Rate:** 0.5 [S2].

### Direct 3D Generation
A broader category of methods that learn a feedforward or diffusion model to output Gaussian fields directly from text or images, without an explicit intermediate image set [S3]. This amortizes reconstruction cost and promotes multi-view coherence [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Alleviates the Janus problem; mitigates floaters; bypasses iterative per-scene optimization [S1].
*   **Limitations:** Requires training a diffusion model; relies on optimal transport for representation [S1].

## Background
GaussianDiffusion was proposed in [251] (cited in [S1]) as a method to improve 3D generation quality by incorporating structured noises from multiple viewpoints and using variational 3DGS [S1]. It is part of a family of methods that use generative priors to predict Gaussian parameters directly, avoiding the limitations of iterative densification [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S3] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
