# GaussianDiffusion

Also called: GaussianCube

## Overview
GaussianDiffusion is a method that trains a 3D diffusion model based on a GaussianCube representation to synthesize 3D Gaussian fields directly [S1]. It solves the problem of multi-view inconsistency (the "Janus problem") and the generation of spurious geometry ("floaters") by incorporating structured noises from multiple viewpoints and using variational 3DGS [S1]. It sets itself apart from iterative optimization approaches by amortizing reconstruction cost through learned priors, enabling near-real-time sampling at test time [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 2D images or text prompts (implied by diffusion context) [S1].
*   **Output:** A set of 3D Gaussians defined by position, covariance, opacity, and spherical harmonics coefficients [S1].

**Data Structures:**
*   **GaussianCube:** A voxelized representation of a constant number of Gaussians. The conversion from the continuous Gaussian field to this grid is performed via optimal transport [S1].
*   **Diffusion Model:** A neural network trained to denoise the GaussianCube representation [S1].

**Steps:**
1.  **Representation Conversion:** Convert a constant number of 3D Gaussians into a GaussianCube representation using optimal transport voxelization [S1].
2.  **Diffusion Training:** Train a 3D diffusion model on the GaussianCube representations [S1].
3.  **Structured Noise Injection:** During the diffusion process, incorporate structured noises from multiple viewpoints to alleviate the Janus problem [S1].
4.  **Variational Optimization:** Use variational 3DGS principles during generation to mitigate floaters and improve generation results [S1].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   The difference between the diffusion prior and the training process of the diffusion model can impair the quality of 3D generation [S1].

## Subtopics

### [Direct 3D Generation](11810-direct-3d-generation.md)
Direct 3D Generation is a method that learns a feedforward or diffusion model to output Gaussian fields directly from text or images, without an explicit intermediate image set. It solves the problem of high reconstruction latency and multi-view inconsistency inherent in iterative optimization approaches by amortizing the reconstruction cost through learned priors. Unlike iterative methods that optimize per-scene parameters, this approach enables near-real-time sampling at test time and naturally promotes multi-view coherence.

### Gaussian Masking
Gaussian Masking is a technique used to prune Gaussians at every densification step and for every 500 iterations after densification stops [S2]. It utilizes a Gaussian masking loss with a weight $\lambda_m = 0.05$ [S2]. The threshold for Gaussian masks is $\epsilon_m = 0.05$ [S2]. The learning rate for Gaussian masks is set to 0.5 [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Alleviates the Janus problem through structured noise [S1]. Mitigates floaters via variational 3DGS [S1]. Enables near-real-time sampling at test time [S1].
*   **Limitations:** The difference between the diffusion prior and the training process can impair quality [S1].
*   **When to use:** When direct synthesis from text/images is required without iterative per-scene optimization [S1].
*   **When not to use:** When iterative optimization with explicit priors (like LoRA or classifier-free guidance adjustments) is preferred to bridge the gap between prior and training [S1].

## Background
GaussianDiffusion was proposed in reference [251] cited in [S1]. It is part of the broader effort to improve 3D generation quality by addressing the Janus problem and floaters. It is related to other methods such as ISM [S1], iterative optimization of 3D models and diffusion priors [S1], and GaussianCube [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
