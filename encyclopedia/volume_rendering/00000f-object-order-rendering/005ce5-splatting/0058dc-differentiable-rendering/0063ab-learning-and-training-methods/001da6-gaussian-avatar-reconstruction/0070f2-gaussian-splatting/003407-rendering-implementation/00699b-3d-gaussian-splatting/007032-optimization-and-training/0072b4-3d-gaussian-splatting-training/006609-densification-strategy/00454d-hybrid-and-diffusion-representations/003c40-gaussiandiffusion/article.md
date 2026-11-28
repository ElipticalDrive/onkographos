# GaussianDiffusion

Also called: GaussianDiffusion [S1][S2, unofficial]

## Overview
GaussianDiffusion is a method that trains a 3D diffusion model based on a GaussianCube representation to generate 3D Gaussian fields directly from conditioning signals [S1][S2, unofficial]. It solves the problem of multi-view inconsistency (the "Janus problem") and the generation of floating artifacts common in single-view diffusion priors by incorporating structured noise from multiple viewpoints and using variational 3DGS [S1][S2, unofficial]. Unlike iterative optimization methods, it amortizes the generation cost over a trained network, enabling near-real-time sampling at test time [S1].

## How it works
**Inputs:** Conditioning signal (e.g., text or image) and structured noise from multiple viewpoints.
**Outputs:** A set of 3D Gaussians defined by attributes $\mu_k, s_k, q_k, \alpha_k, \mathbf{c}_k$.

**Data Structures:**
The method operates on a **GaussianCube** representation. This is a voxelized grid where each voxel contains a constant number of Gaussians. The conversion from a fixed set of Gaussians to the GaussianCube is performed via optimal transport [S1].

**Steps:**
1.  **Representation Conversion:** Convert the target constant number of 3D Gaussians into a GaussianCube representation using optimal transport voxelization [S1].
2.  **Diffusion Training:** Train a 3D diffusion model on the GaussianCube representation. The model learns to denoise the Gaussian attributes conditioned on the input signal [S1].
3.  **Structured Noise Incorporation:** During training and inference, incorporate structured noise from multiple viewpoints into the diffusion process [S1][S2, unofficial]. This step is critical to alleviate the Janus problem, ensuring multi-view consistency [S1][S2, unofficial].
4.  **Variational 3DGS Integration:** Use variational 3DGS within the generation pipeline to mitigate the appearance of floaters (artifacts) in the generated scene [S1][S2, unofficial].

**Parameters:**
*   Specific hyperparameters for the diffusion model (e.g., noise schedule, number of steps) are not given in the sources.
*   The specific implementation of the optimal transport algorithm for voxelization is not detailed in the sources.

**Edge Cases and Pitfalls:**
*   **Janus Problem:** Without structured noise from multiple viewpoints, the generated 3D object may exhibit inconsistent features across different views [S1][S2, unofficial].
*   **Floaters:** Standard 3DGS generation can produce floating artifacts; variational 3DGS is employed to mitigate this [S1][S2, unofficial].

## Subtopics

### [Direct 3D Generation](11810-direct-3d-generation.md)
Direct 3D Generation refers to methods that learn a feedforward or diffusion model to output Gaussian fields directly from text or images, without an explicit intermediate image set. It solves the problem of high reconstruction latency and multi-view inconsistency inherent in iterative optimization by amortizing the cost over a trained network. Unlike iterative reconstruction, it naturally promotes multi-view coherence and enables near-real-time sampling at test time.

### Gaussian Masking
Gaussian Masking is a technique used to prune Gaussians during the training process to improve efficiency and quality [S3].
*   **Step:** Use Gaussian masks to prune Gaussians at every densification step and for every 500 iterations after densification stops [S3].
*   **Parameters:**
    *   $\lambda_m$: Loss weight for Gaussian masking loss. Value is 0.05 [S3].
    *   $\epsilon_m$: Threshold for Gaussian masks. Value is 0.05 [S3].
    *   Learning rate for Gaussian masks: 0.5 [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Alleviates the Janus problem through structured multi-view noise; mitigates floaters via variational 3DGS; enables near-real-time generation at test time [S1][S2, unofficial].
*   **Limitations:** Requires training a diffusion model on a GaussianCube representation, which may have higher initial setup costs compared to iterative methods [S1].
*   **When to use:** When multi-view consistency is critical and near-real-time generation is required [S1].
*   **When not to use:** When iterative optimization is acceptable and multi-view consistency is less of a concern [S1].

## Background
GaussianDiffusion was proposed by [251] (cited in [S1][S2, unofficial]). It is part of the broader effort to improve 3D generation quality by addressing specific issues like the Janus problem and floaters in 3D Gaussian Splatting [S1][S2, unofficial]. The method builds on the GaussianCube representation, which converts a constant number of Gaussians into a voxelized format via optimal transport [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S3] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
