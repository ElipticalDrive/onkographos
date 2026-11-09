# GaussianDiffusion

Also called: GaussianCube

## Overview
GaussianDiffusion is a method that trains a 3D diffusion model based on a **GaussianCube** representation [S1]. It addresses the "Janus problem" (inconsistent geometry across viewpoints) and the generation of "floaters" (spurious, low-opacity primitives) common in 3D Gaussian Splatting (3DGS) generation [S1]. It sets itself apart from standard 3DGS training by incorporating structured multi-view noise into the diffusion process and using variational 3DGS principles to mitigate floaters [S1].

## How it works
**Inputs:** A set of 3D Gaussians $\mathcal{G}$ representing the scene.
**Outputs:** A trained 3D diffusion model capable of generating GaussianCube representations.

**Data Structures:**
The core representation is the **GaussianCube**. It is derived from a constant number of Gaussians. The conversion involves voxelization via optimal transport [S1]. The specific layout of the GaussianCube tensor is not detailed in the sources, but it serves as the latent space for the diffusion model.

**Steps:**
1.  **Representation Conversion:** Convert the initial set of 3D Gaussians into the GaussianCube representation. This is achieved by voxelizing the Gaussians using optimal transport to ensure a structured, constant-size input for the diffusion model [S1].
2.  **Diffusion Training:** Train a 3D diffusion model on the GaussianCube representations.
3.  **Structured Noise Injection:** During the diffusion process, incorporate structured noises from multiple viewpoints. This specific noise scheduling is designed to alleviate the Janus problem by ensuring consistency across different camera angles [S1].
4.  **Variational Optimization:** Use variational 3DGS principles during the generation or refinement phase. This helps mitigate the appearance of floaters (spurious primitives) by better modeling the uncertainty and distribution of the Gaussian parameters [S1].

**Parameters:**
*   Specific hyperparameters for the diffusion model (e.g., noise schedule, number of steps) are not given in the sources.
*   The number of Gaussians in the initial set is constant, but the exact count is not specified [S1].

**Edge Cases and Pitfalls:**
*   **Janus Problem:** Without structured multi-view noise, generated models may exhibit conflicting geometry from different viewpoints [S1].
*   **Floaters:** Standard 3DGS generation often produces low-opacity, spurious primitives. The variational approach is specifically employed to mitigate this [S1].

## Subtopics

### Gaussian Masking
A variant technique used to prune Gaussians. It applies Gaussian masks to remove primitives at every densification step and for 500 iterations after densification stops [S2].
*   **Loss Weight ($\lambda_m$):** 0.05 [S2].
*   **Threshold ($\epsilon_m$):** 0.05 [S2].
*   **Learning Rate:** 0.5 [S2].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Alleviates the Janus problem and reduces floaters in generated 3DGS models [S1].
*   **Limitations:** Requires the conversion of Gaussians to a GaussianCube via optimal transport, which adds computational overhead during the representation preparation phase [S1].

## Background
GaussianDiffusion was proposed in the context of improving 3D generation from 2D images using 3D Gaussian Splatting [S1]. It is cited as reference [251] in "Recent advances in 3D Gaussian splatting" [S1]. The method is part of a broader effort to improve the quality of diffusion-based 3D generation by addressing specific artifacts like the Janus problem and floaters [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
