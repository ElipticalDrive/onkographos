# GaussianDiffusion

Also called: GaussianDiffusion [S1]

## Overview
GaussianDiffusion is a method that trains a 3D diffusion model based on a GaussianCube representation to generate 3D Gaussian primitives directly from conditioning signals [S1][S2, unofficial]. It solves the problem of multi-view inconsistency (the "Janus problem") and the generation of spurious artifacts ("floaters") common in single-view 3D generation [S1][S2, unofficial]. Unlike iterative optimization methods, it amortizes the generation cost over a trained network, enabling near-real-time sampling [S1][S2, unofficial]. It is distinguished from its sibling [TriplaneGaussian](7558-triplanegaussian.md) by its use of a voxelized GaussianCube representation converted via optimal transport, rather than tri-plane grids [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Conditioning signal (e.g., text or image) and noise.
*   **Output:** A set of 3D Gaussians $\mathcal{G}$, represented implicitly via a GaussianCube.

**Data Structures:**
*   **GaussianCube:** A voxelized representation of a constant number of Gaussians. The conversion from the discrete Gaussian set to this structured grid is performed via optimal transport [S1][S2, unofficial].
*   **Diffusion Model:** A neural network that predicts noise or scores in the GaussianCube latent space.

**Steps:**
1.  **Representation Conversion:** Convert a constant number of 3D Gaussians into a GaussianCube representation using optimal transport [S1][S2, unofficial].
2.  **Diffusion Training:** Train a 3D diffusion model on the GaussianCube representations.
3.  **Structured Noise Incorporation:** Incorporate structured noise from multiple viewpoints during the diffusion process to alleviate the Janus problem [S1][S2, unofficial].
4.  **Variational 3DGS:** Utilize Variational 3DGS principles during generation to mitigate floaters and improve generation results [S1][S2, unofficial].
5.  **Generation:** Sample from the trained diffusion model to produce the GaussianCube, which is then decoded into the final set of 3D Gaussians [S1][S2, unofficial].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Janus Problem:** Without structured noise from multiple viewpoints, the model may generate inconsistent geometry from different angles [S1][S2, unofficial].
*   **Floaters:** Spurious Gaussian primitives may appear; mitigated by using Variational 3DGS [S1][S2, unofficial].

## Subtopics

### [Direct 3D Generation](11810-direct-3d-generation.md)
Direct 3D Generation refers to methods that learn a feedforward or diffusion model to output Gaussian fields directly from text or images, without an explicit intermediate image set. It solves the problem of high reconstruction latency and multi-view inconsistency inherent in iterative optimization by amortizing the cost over a trained network. Unlike iterative reconstruction, it naturally promotes multi-view coherence and enables near-real-time sampling at test time.

### Gaussian Masking
Gaussian Masking is a technique used to prune Gaussians during the training process [S3]. It is applied at every densification step and for every 500 iterations after densification stops [S3].
*   **Parameters:**
    *   $\lambda_m$: Loss weight for Gaussian masking loss. Value is 0.05 [S3].
    *   $\epsilon_m$: Threshold for Gaussian masks. Value is 0.05 [S3].
    *   Learning rate for Gaussian masks: 0.5 [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Alleviates the Janus problem through structured multi-view noise; mitigates floaters via Variational 3DGS; enables near-real-time generation [S1][S2, unofficial].
*   **Limitations:** Requires training a diffusion model; relies on optimal transport for representation conversion [S1][S2, unofficial].
*   **When to use:** When near-real-time 3D generation from text or images is required, and multi-view consistency is critical [S1][S2, unofficial].
*   **When not to use:** When iterative optimization is acceptable and high-fidelity control over individual Gaussians is needed without diffusion artifacts [S1][S2, unofficial].

## Background
GaussianDiffusion was proposed in the context of recent advances in 3D Gaussian splatting [S1][S2, unofficial]. It addresses limitations in single-view 3D generation, such as the Janus problem and floaters, by integrating diffusion models with a GaussianCube representation [S1][S2, unofficial]. The method builds on the principles of hybrid and diffusion representations, where Gaussian attributes are predicted directly from conditioning signals [Parent]. It is related to other methods like LucidDreamer, which uses interval score matching, and GVGEN, which uses a 3D Gaussian volume representation [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S3] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
