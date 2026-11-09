# GaussianDiffusion
## Overview
GaussianDiffusion is a training strategy for 3D Gaussian Splatting (3DGS) that integrates diffusion priors to improve generation quality, specifically targeting the "Janus problem" (inconsistent geometry across viewpoints) and the presence of "floaters" (spurious, low-opacity artifacts). It distinguishes itself from standard densification methods by incorporating structured noise from multiple viewpoints and employing variational 3DGS principles to mitigate floaters during the optimization process [S1].

## How it works
GaussianDiffusion operates within the differentiable training loop of 3D Gaussian Splatting. It modifies the standard densification and rendering pipeline by introducing diffusion-based supervisions and variational constraints.

**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G}_i$ with parameters (mean $\mu_i$, covariance $\Sigma_i$, opacity $\alpha_i$, spherical harmonics coefficients), multi-view images, and camera poses.
*   **Output:** Optimized 3D Gaussian parameters and rendered images consistent across viewpoints.

**Data Structures:**
The method relies on the standard 3DGS Gaussian representation. Additionally, it utilizes a diffusion model (e.g., Stable Diffusion) as a prior. The specific internal structure of the diffusion model is external to the Gaussian set but provides gradient signals.

**Steps:**
1.  **Structured Noise Injection:** During training, structured noise is incorporated from multiple viewpoints. This noise is not random but aligned with the geometric structure to guide the Gaussian distribution towards consistent 3D shapes, reducing view-dependent inconsistencies (Janus effect) [S1].
2.  **Variational 3DGS Formulation:** The method employs a variational approach to 3DGS. This involves treating the Gaussian parameters as latent variables in a probabilistic framework. The loss function includes a term that encourages the Gaussian distribution to match the prior implied by the diffusion model, effectively regularizing the opacity and position to reduce floaters [S1].
3.  **Diffusion Prior Integration:** The rendering pipeline is coupled with a pre-trained diffusion model. The rendered images are used as inputs to the diffusion model (or vice versa, depending on the specific implementation of SDS or similar scores), providing gradients that enforce photorealistic textures and geometric consistency [S1].
4.  **Densification and Pruning:** Standard densification strategies (splitting, cloning, pruning) are applied based on gradient norms and opacity, but the variational constraints help prevent the proliferation of low-opacity floaters that typically require aggressive pruning [S1].

**Formulas:**
While the exact mathematical formulation of the variational loss is not detailed in the provided sources, the core concept involves minimizing a loss $\mathcal{L}$ composed of:
$$ \mathcal{L} = \mathcal{L}_{render} + \lambda_{diff} \mathcal{L}_{diff} + \lambda_{var} \mathcal{L}_{var} $$
Where $\mathcal{L}_{render}$ is the standard reconstruction loss, $\mathcal{L}_{diff}$ is the diffusion prior loss (e.g., Score Distillation Sampling), and $\mathcal{L}_{var}$ is the variational regularization term that mitigates floaters [S1].

**Parameters:**
*   **Diffusion Model:** Not specified in the sources. Typically a pre-trained text-to-image or image-to-image diffusion model.
*   **Variational Weight:** Not specified in the sources.
*   **Noise Schedule:** Structured noise is applied, but the specific schedule is not detailed [S1].

**Edge Cases and Pitfalls:**
*   **Janus Problem:** Without structured noise, multi-view consistency is poor [S1].
*   **Floaters:** Standard 3DGS often produces low-opacity artifacts. Variational constraints are necessary to mitigate these [S1].
*   **Computational Cost:** Integrating diffusion models adds significant computational overhead compared to standard 3DGS training [S1].

## Subtopics

### Gaussian Masking
Gaussian Masking is a variant technique that uses learnable Gaussian masks to prune Gaussians. It operates at every densification step and for every 500 iterations after densification stops. It introduces a loss weight $\lambda_m = 0.05$, a threshold $\epsilon_m = 0.05$, and a learning rate of 0.5 for the Gaussian masks [S2].

## Operating characteristics
*   **Time Complexity:** Higher than standard 3DGS due to the integration of diffusion model inference and variational calculations.
*   **Memory Complexity:** Standard 3DGS memory usage plus overhead for diffusion model states and mask parameters.
*   **Strengths:** Effectively alleviates the Janus problem and reduces floaters, leading to higher quality 3D generation [S1].
*   **Limitations:** Increased training time and complexity. Requires careful tuning of variational and diffusion weights.
*   **When to Use:** When high-fidelity 3D generation from multi-view images is required, and view consistency is critical.
*   **When Not to Use:** When training speed is paramount and minor view inconsistencies are acceptable.

## Background
GaussianDiffusion was proposed to address limitations in standard 3D Gaussian Splatting for 3D generation tasks. Specifically, it aims to solve the Janus problem and reduce floaters by incorporating structured noise and variational principles [S1]. It builds upon the foundation of 3DGS and diffusion-based 3D generation methods like SDS [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
