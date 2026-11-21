# Diffusion Models (DDPMs)
Also called: Denoising Diffusion Probabilistic Models

## Overview
Diffusion Models (DDPMs) generate data by learning to reverse a gradual noising process, iteratively denoising from pure noise to a data distribution sample [S1, unofficial]. In the context of volume rendering, they are applied to volume generation, Transfer Function (TF) design, and visualization synthesis [S1, unofficial]. They are distinguished from GAN-based siblings (e.g., [Opacity GAN](7664-opacity-gan.md)) by their iterative refinement nature, which naturally handles the multimodal distributions common in scientific data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Pure Gaussian noise $\mathbf{x}_T \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$ and a diffusion time step $t$.
*   **Output:** A denoised sample $\mathbf{x}_0$ representing the target data (e.g., volume features or transfer function parameters).

**Data Structures:**
*   **Diffusion Schedule:** Arrays $\alpha_t$ and $\sigma_t$ defining the noise level at step $t$.
*   **Denoising Network:** A neural network $f_\phi$ with parameters $\phi$ that predicts the clean signal or noise.

**Steps:**
1.  **Forward Process (Training):** Given a clean data point $\mathbf{x}_0$, add Gaussian noise $\boldsymbol{\epsilon} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$ at step $t$ to create a noisy sample $\mathbf{x}_t$. The specific formulation for latent codes is $\mathbf{x}_t^{(i)} = \alpha^{(t)} \mathbf{x}_i + \sigma^{(t)} \boldsymbol{\epsilon}$ [S2].
2.  **Denoising Prediction:** The network $f_\phi(\mathbf{x}_t, t)$ predicts the denoised code $\hat{\mathbf{x}}_\phi(\mathbf{x}_t, t)$ [S2].
3.  **Loss Calculation:** Minimize the simplified L2 denoising loss:
    $$ \mathcal{L}_{\text{diff}}(\phi) = \mathbb{E}_{i,t,\boldsymbol{\epsilon}} \left[ \frac{1}{2} \|\hat{\mathbf{x}}_\phi(\mathbf{x}_i^{(t)}, t) - \mathbf{x}_i^{(t)}\|^2 \right] $$
    where $t \sim U(0, T)$ and $w^{(t)}$ is an empirical time-dependent weighting function [S2].
4.  **Sampling:** Starting from random Gaussian noise $\mathbf{x}_T$, recursively denoise using a solver (e.g., DDIM) until reaching $\mathbf{x}_0$ [S2].

**Parameters:**
*   $T$: Total number of diffusion steps.
*   $t$: Diffusion time step, sampled uniformly from $U(0, T)$ [S2].
*   $w^{(t)}$: Empirical time-dependent weighting function [S2].

**Edge Cases and Pitfalls:**
*   Reconstruction from sparse views is exceptionally difficult without a learned prior [S2].
*   The method requires a two-stage training scheme for latent diffusion: first training an auto-decoder to obtain per-scene latent codes, then training the LDM on those codes [S2].

## Subtopics

### Latent Diffusion Models (LDM) for 3D
A model that learns a prior distribution $p_\phi(\mathbf{x})$ in the latent space to enable expressive latent representations, typically trained in two stages for NeRF auto-decoders [S2]. It injects Gaussian perturbation $\boldsymbol{\epsilon} \sim \mathcal{N}(\mathbf{0}, \mathbf{I})$ into the code $\mathbf{x}_i$ to yield a noisy code $\mathbf{x}_i^{(t)} = \alpha^{(t)} \mathbf{x}_i + \sigma^{(t)} \boldsymbol{\epsilon}$ at diffusion time step $t$ [S2]. It is chosen when 3D generation or reconstruction from sparse views is required, leveraging the expressiveness of latent grids [S2].

## Operating characteristics
*   **Strengths:** Naturally handles multimodal distributions common in scientific data [S1, unofficial]. Enables holistic unification of 3D generation and reconstruction [S2].
*   **Limitations:** Requires significant computational resources for iterative denoising. Sparse view reconstruction is difficult without strong priors [S2].
*   **Performance:** Not given in the sources.

## Background
Diffusion models entered volume visualization after Wang and Han’s survey [147] [S1, unofficial]. They are part of the broader deep learning evolution in scientific visualization, following CNN-based and GAN-driven frameworks [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S2] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
