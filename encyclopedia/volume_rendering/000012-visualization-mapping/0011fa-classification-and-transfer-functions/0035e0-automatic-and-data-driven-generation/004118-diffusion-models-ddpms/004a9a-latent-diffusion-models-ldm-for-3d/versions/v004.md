# Latent Diffusion Models (LDM) for 3D

Also called: 3D Latent Diffusion, LDM for 3D, Latent Diffusion Models (LDMs)

## Overview
Latent Diffusion Models (LDM) for 3D perform diffusion in a latent space, significantly improving computational efficiency and output quality [S1]. In the context of 3D volume rendering, hierarchical LDMs define a generative model over latent variables representing scene geometry, camera trajectories, and fine details, enabling the synthesis of voxel-based neural fields [S2]. This approach sets it apart from standard Denoising Diffusion Probabilistic Models (DDPMs) by performing diffusion in a latent space rather than pixel space, allowing for stable and efficient generation of complex 3D scenes conditioned on hierarchical inputs [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Training Input:** A dataset of scene latent codes $\{x_i\}$ extracted from a pre-trained auto-decoder (e.g., a NeRF auto-decoder) [S2].
*   **Training Output:** A denoising network with parameters $\phi$ that predicts the clean latent code from a noisy version [S2].
*   **Inference Input:** Random Gaussian noise $x^{(T)}$ and optionally conditioning signals (e.g., text, single-view images) [S1, S2].
*   **Inference Output:** A clean latent code $x^{(0)}$ representing the 3D scene, which is then decoded into a neural field (e.g., NeRF) or mesh [S2].

**Data Structures:**
*   **Latent Code ($x_i$):** A compressed representation of the 3D scene [S2]. In methods like SSDNeRF, this is often a set of triplane features [S2].
*   **Denoising Network:** A neural network parameterized by $\phi$ [S2]. It takes a noisy latent code $x^{(t)}$ and time step $t$ as input and outputs a predicted clean code $\hat{x}_\phi$ [S2].

**Algorithm Steps:**

1.  **Pre-training (Auto-decoder):** Train a scene auto-decoder to map 3D scenes to latent codes $x_i$ [S2]. This is typically done by minimizing a variational upper bound on the negative log-likelihood of observed data, combining a rendering loss and a prior term [S2].
    $$ L = \mathbb{E}_i [-\log p_\psi(\{y_{ij}^g\}|x_i, \{r_{ij}\})] + \mathbb{E}_i [-\log p_\phi(x_i)] $$
    where the first term is the rendering loss and the second is the prior term [S2].

2.  **Latent Extraction:** Extract the per-scene latent codes $x_i$ from the trained auto-decoder [S2]. These codes are treated as the "real data" for the diffusion model [S2].

3.  **Diffusion Process (Forward):** For a given latent code $x_i$ and diffusion time step $t$, inject Gaussian noise to create a noisy code $x_i^{(t)}$:
    $$ x_i^{(t)} = \alpha^{(t)} x_i + \sigma^{(t)} \epsilon $$
    where $\epsilon \sim \mathcal{N}(0, I)$ is Gaussian noise, and $\alpha^{(t)}, \sigma^{(t)}$ are empirical noise schedule functions [S2].

4.  **Denoising Network Training:** Train the denoising network $\hat{x}_\phi$ to predict the clean latent code $x_i$ from the noisy code $x_i^{(t)}$ [S2]. The network is optimized using a simplified L2 denoising loss:
    $$ \mathcal{L}_{diff}(\phi) = \mathbb{E}_{i,t,\epsilon} \left[ \frac{1}{2} w^{(t)} \|\hat{x}_\phi(x_i^{(t)}, t) - x_i^{(t)}\|^2 \right] $$
    where $t \sim \mathcal{U}(0, T)$ is sampled uniformly, and $w^{(t)}$ is an empirical time-dependent weighting function [S2].

5.  **Sampling (Inference):** To generate a new 3D scene, start with random Gaussian noise $x^{(T)}$ and iteratively denoise it using a solver (e.g., DDIM) to obtain $x^{(0)}$ [S2].
    *   **Unconditional Generation:** Sample directly from the prior $p_\psi(x)$ [S2].
    *   **Guided Reconstruction:** Guide the sampling process using gradients of the rendering loss against known observations (e.g., input images) to reconstruct a specific scene [S2].

**Parameters:**
*   $T$: Total number of diffusion steps. Not given in the sources.
*   $t$: Diffusion time step, sampled uniformly from $\mathcal{U}(0, T)$ [S2].
*   $w^{(t)}$: Empirical time-dependent weighting function. Not given in the sources.
*   $\alpha^{(t)}, \sigma^{(t)}$: Empirical noise schedule functions. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The sources do not specify precision requirements. [G] Standard practice suggests using FP16 or BF16 for training to save memory, but FP32 may be needed for stability in the denoising network.
*   **Boundaries:** The noise schedule functions $\alpha^{(t)}$ and $\sigma^{(t)}$ must be well-defined for $t \in [0, T]$. [G] Common schedules include linear or cosine schedules.
*   **Degenerate Input:** If the auto-decoder fails to converge, the extracted latent codes $x_i$ may not form a valid distribution for the diffusion model. [G] Careful validation of the auto-decoder is required.

## Subtopics

### [Latent Diffusion Models (LDM)](14900-latent-diffusion-models-ldm.md)
Latent Diffusion Models (LDMs) perform the diffusion process in a compressed latent space rather than the high-dimensional pixel space, significantly improving computational efficiency and output quality. In the context of 3D volume rendering, hierarchical LDMs define a generative model over latent variables representing scene geometry, camera trajectories, and fine details, enabling the synthesis of voxel-based neural fields. This approach sets it apart from standard Denoising Diffusion Probabilistic Models (DDPMs) by decoupling the diffusion process from the expensive decoding step, allowing for stable and efficient generation of complex 3D scenes conditioned on hierarchical inputs.

## Operating characteristics
*   **Time and Memory Complexity:** Significantly more efficient than DDPMs in pixel space due to the reduced dimensionality of the latent space [S1].
*   **Measured Performance:** Not given in the sources [S1, S2].
*   **Strengths:** High computational efficiency, improved output quality, ability to handle complex 3D scenes, and support for conditional generation (e.g., text, single-view) [S1, S2].
*   **Limitations:** Requires a pre-trained auto-decoder, which adds complexity to the training pipeline. The quality of the generated 3D scenes is dependent on the expressiveness of the latent representation [S2].
*   **When to Use:** When generating high-quality 3D scenes is required, and computational resources are limited compared to pixel-space diffusion [S1]. Also suitable for tasks requiring conditional generation or reconstruction from sparse views [S2].
*   **When Not to Use:** When the scene geometry is extremely simple and can be generated efficiently with DDPMs, or when a pre-trained auto-decoder is not available and training one is prohibitive [G].

## Background
Latent Diffusion Models (LDMs) were introduced to address the computational inefficiency of DDPMs in high-dimensional spaces [S1]. In the context of 3D, LDMs have been adapted to generate neural fields (e.g., NeRFs) by learning a prior distribution over latent codes [S2]. The Single-Stage Diffusion NeRF (SSDNeRF) framework is a notable example that unifies 3D generation and reconstruction by jointly learning a triplane NeRF auto-decoder and a triplane latent diffusion prior [S2].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
