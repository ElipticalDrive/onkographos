# GaussianDiffusion

Also called: GaussianCube

## Overview
GaussianDiffusion is a method that trains a 3D diffusion model based on a **GaussianCube** representation to synthesize 3D Gaussian fields directly [S1]. It addresses the "Janus problem" (multi-view inconsistency) by incorporating structured noises from multiple viewpoints and mitigates the generation of "floaters" (spurious geometry) through variational 3DGS [S1]. It sets itself apart from standard 3DGS training by bypassing iterative densification loops, instead using a voxelized intermediate representation converted via optimal transport [S1].

## How it works
**Inputs:** A dataset of 3D scenes or images; a target number of Gaussians $N$.
**Outputs:** A set of 3D Gaussians $\{G_i\}_{i=1}^N$ defined by position $\mathbf{x}_i$, covariance $\Sigma_i$, color $\mathbf{c}_i$, and opacity $\alpha_i$.

**Data Structures:**
1.  **GaussianCube:** A voxel grid representation of the Gaussian field. Each voxel stores aggregated Gaussian attributes.
2.  **Diffusion Model:** A neural network $p_\theta(\mathbf{z}_t | \mathbf{z}_{t-1})$ or $p_\theta(\mathbf{z}_0 | \mathbf{z}_t)$ operating in the latent space of the GaussianCube.

**Steps:**
1.  **GaussianCube Construction:** Convert a constant number of Gaussians into a voxel grid using optimal transport [S1]. This involves mapping the continuous Gaussian parameters to discrete voxel bins while preserving statistical moments or density mass.
2.  **Structured Noise Injection:** During the diffusion forward process, incorporate structured noise derived from multiple viewpoints [S1]. This ensures that the noise pattern respects multi-view geometric consistency, reducing the Janus problem.
3.  **Diffusion Training:** Train the diffusion model to predict the noise or the clean GaussianCube from noisy inputs. The loss function typically includes a variational bound to regularize the generated Gaussians, mitigating floaters [S1].
4.  **Sampling:** Use the trained diffusion model (e.g., DDIM inversion or iterative denoising) to generate a clean GaussianCube from random noise or a text/image condition.
5.  **Un-voxelization:** Convert the generated GaussianCube back into a set of explicit 3D Gaussians for rendering [S1].

**Parameters:**
*   Number of Gaussians $N$: Constant, determined by the resolution of the GaussianCube.
*   Diffusion steps: Standard for the chosen diffusion sampler (e.g., DDIM).
*   Variational regularization weight: Not given in the sources.

**Edge Cases:**
*   **Precision:** Optimal transport calculations can be computationally expensive and sensitive to the discretization of the voxel grid [S1].
*   **Boundaries:** The conversion between Gaussian space and voxel space may introduce artifacts at the boundaries of the voxel grid if not handled with smooth aggregation [S1].

## Subtopics

### Gaussian Masking
A technique used to prune Gaussians during training. It applies Gaussian masks to prune Gaussians at every densification step and for every 500 iterations after densification stops [S2]. It uses a loss weight $\lambda_m = 0.05$, a threshold $\epsilon_m = 0.05$, and a learning rate of 0.5 for the Gaussian masks [S2]. This helps reduce the number of Gaussians without significantly degrading rendering quality [S2].

### Direct 3D Generation
A broader category of methods that learn a feedforward or diffusion model to output Gaussian fields directly from text or images, without an explicit intermediate image set [S3]. These methods amortize reconstruction cost and promote multi-view coherence [S3]. They face challenges in the high-dimensional output space and require large 3D training corpora [S3].

## Operating characteristics
*   **Time Complexity:** Training time depends on the diffusion model size and the number of diffusion steps. The optimal transport step for GaussianCube construction adds overhead [S1].
*   **Memory Complexity:** Requires memory for the voxel grid (GaussianCube) and the diffusion model parameters [S1].
*   **Performance:** Not given in the sources.
*   **Strengths:** Alleviates the Janus problem and reduces floaters [S1]. Bypasses slow iterative densification [S1].
*   **Limitations:** Requires complex voxelization and optimal transport steps [S1]. Dependent on the quality of the diffusion prior [S1].

## Background
GaussianDiffusion was proposed in the context of recent advances in 3D Gaussian splatting [S1]. It builds on the idea of using diffusion models for 3D generation, adapting them to the specific structure of 3D Gaussians via the GaussianCube representation [S1]. The method aims to improve generation quality by addressing specific issues like multi-view inconsistency and spurious geometry [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S3] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
