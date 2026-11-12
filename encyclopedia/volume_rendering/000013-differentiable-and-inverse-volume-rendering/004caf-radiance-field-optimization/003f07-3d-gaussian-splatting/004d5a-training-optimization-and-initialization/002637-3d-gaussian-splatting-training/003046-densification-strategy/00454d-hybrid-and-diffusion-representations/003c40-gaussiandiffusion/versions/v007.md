# GaussianDiffusion

Also called: GaussianCube

## Overview
GaussianDiffusion is a method that trains a 3D diffusion model based on a **GaussianCube** representation to generate 3D Gaussian fields directly [S1]. It addresses the "Janus problem" (inconsistent geometry from different viewpoints) by incorporating structured noises from multiple viewpoints and mitigates "floaters" (spurious Gaussians) by using variational 3DGS [S1]. Unlike sibling methods that may rely on iterative optimization or intermediate image sets, GaussianDiffusion maps a compact input to an explicit 3D Gaussian field $\mathcal{G}$ in a single forward pass or sampling step, leveraging the structured nature of the GaussianCube [S1, S3].

## How it works
**Inputs:** A constant number of 3D Gaussians $\mathcal{G}_{init} = \{ \mu_i, \Sigma_i, \alpha_i, c_i \}_{i=1}^{N}$ representing a 3D object or scene.
**Outputs:** A generated 3D Gaussian field $\mathcal{G}_{gen}$ via the diffusion process.

**Data Structures:**
1.  **GaussianCube ($\mathcal{C}$):** A voxelized representation of the Gaussian field.
    *   Layout: A 3D grid of voxels. Each voxel stores aggregated Gaussian attributes (mean, covariance, opacity, color) mapped via optimal transport.
    *   Size: Determined by the resolution of the voxel grid and the fixed number of Gaussians $N$ used for the initial conversion.
2.  **Diffusion Model ($\epsilon_\theta$):** A neural network that predicts noise added to the GaussianCube representation at various timesteps $t$.

**Steps:**
1.  **GaussianCube Conversion:** Convert the initial set of $N$ Gaussians into the GaussianCube representation $\mathcal{C}$ using voxelization via optimal transport [S1]. This creates a structured, grid-based latent space suitable for 3D diffusion.
2.  **Structured Noise Injection:** During the forward diffusion process, add noise to the GaussianCube. To alleviate the Janus problem, the noise is structured to incorporate information from multiple viewpoints [S1].
3.  **Diffusion Training:** Train the noise prediction model $\epsilon_\theta$ to reverse the diffusion process. The model learns to predict the noise $\epsilon$ added to the GaussianCube at timestep $t$ given the noisy cube $\mathcal{C}_t$ and potentially conditioning information (e.g., text or image) [S1].
4.  **Variational Regularization:** Use variational 3DGS principles during training to better generate results by mitigating floaters [S1]. This involves optimizing the Gaussian parameters to be consistent with the diffusion prior.
5.  **Sampling:** To generate a new 3D Gaussian field, start with random noise in the GaussianCube space and iteratively denoise it using the trained model $\epsilon_\theta$ until a clean GaussianCube is obtained.
6.  **Unvoxelization:** Convert the generated GaussianCube back into a set of explicit 3D Gaussians $\mathcal{G}_{gen}$ [S1].

**Parameters:**
*   Number of Gaussians $N$: Constant number used for the initial conversion. Not given in the sources.
*   Voxel Grid Resolution: Not given in the sources.
*   Diffusion Timesteps: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Janus Problem:** Without structured noise from multiple viewpoints, the generated geometry may be inconsistent across views [S1].
*   **Floaters:** Spurious Gaussians may appear if variational constraints are not properly applied [S1].
*   **Precision:** The conversion via optimal transport requires careful handling of attribute mapping to preserve geometric fidelity [S1].

## Subtopics

### Gaussian Masking
A technique used to prune Gaussians during training. Gaussian masks are applied at every densification step and for every 500 iterations after densification stops [S2].
*   **Loss Weight ($\lambda_m$):** 0.05 [S2].
*   **Threshold ($\epsilon_m$):** 0.05 [S2].
*   **Learning Rate:** 0.5 [S2].

### Direct 3D Generation
A broader category of methods that learn a feedforward or diffusion model to output Gaussian fields directly from text or images, without an explicit intermediate image set [S3]. This amortizes reconstruction cost and promotes multi-view coherence [S3]. GaussianDiffusion falls under this category as it generates 3D Gaussians directly from a diffusion process [S3].

## Operating characteristics
*   **Time Complexity:** Depends on the diffusion sampling steps and the complexity of the GaussianCube voxelization/unvoxelization. Not given in the sources.
*   **Memory Complexity:** Proportional to the size of the GaussianCube grid and the number of Gaussians $N$. Not given in the sources.
*   **Strengths:** Alleviates the Janus problem through structured noise [S1]. Mitigates floaters via variational 3DGS [S1]. Enables direct 3D generation without intermediate image sets [S3].
*   **Limitations:** Requires training a 3D diffusion model, which can be computationally intensive. The quality depends on the GaussianCube representation's ability to capture complex geometries.
*   **When to use:** When generating 3D Gaussian fields directly from text or images with a focus on multi-view consistency and reducing artifacts like floaters [S1, S3].

## Background
GaussianDiffusion was proposed in the context of recent advances in 3D Gaussian Splatting [S1]. It builds on the idea of using diffusion models for 3D generation, adapting them to work with the Gaussian representation [S1]. The method specifically addresses challenges in 3D generation such as the Janus problem and floaters [S1]. It is part of the broader trend of direct 3D generation methods that aim to amortize reconstruction costs [S3].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S3] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
