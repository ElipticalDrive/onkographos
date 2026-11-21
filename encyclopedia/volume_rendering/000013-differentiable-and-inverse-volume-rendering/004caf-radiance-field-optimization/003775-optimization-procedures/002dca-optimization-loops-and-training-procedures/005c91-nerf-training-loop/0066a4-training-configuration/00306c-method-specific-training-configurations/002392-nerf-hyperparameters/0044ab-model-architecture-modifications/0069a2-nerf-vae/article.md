# NeRF-VAE

Also called: Variational Autoencoder for NeRF

## Overview
NeRF-VAE is a generative neural radiance field model that embeds a NeRF within a Variational Autoencoder (VAE) framework to learn a distribution over 3D scenes [S1][S2, unofficial]. It solves the problem of scene-specific training by learning a shared latent space, enabling generalization to out-of-distribution scenes and novel view synthesis from sparse inputs without per-scene optimization [S1][S2, unofficial]. It differs from standard NeRF by replacing the per-scene optimization loop with an amortized inference process using a latent code, and from other generative NeRFs by conditioning the renderer directly on this single global latent vector [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of $N$ input images $\{I_i\}_{i=1}^N$ with known camera poses $\{P_i\}_{i=1}^N$.
- **Output:** A rendered image $\hat{I}$ from a novel view, or a sample from the learned scene distribution.

**Data Structures:**
1.  **Latent Code ($\mathbf{z}$):** A fixed-size vector sampled from a learned distribution $q_\phi(\mathbf{z}|\mathbf{I})$.
2.  **Encoder ($E_\phi$):** A ResNet-based CNN that maps input images to the parameters of the approximate posterior distribution for $\mathbf{z}$ [S1].
3.  **Scene Function ($F_\Theta$):** An attention-based neural network (replacing the standard MLP) that maps spatial coordinates $\mathbf{x}$, viewing directions $\mathbf{d}$, and the latent code $\mathbf{z}$ to density $\sigma$ and color $\mathbf{c}$ [S1].

**Method Steps:**
1.  **Encoding:** Given input images $\mathbf{I}$, the ResNet encoder $E_\phi$ computes the mean $\boldsymbol{\mu}$ and log-variance $\boldsymbol{\sigma}^2$ of the approximate posterior $q_\phi(\mathbf{z}|\mathbf{I}) = \mathcal{N}(\boldsymbol{\mu}, \text{diag}(\boldsymbol{\sigma}^2))$ [S1].
2.  **Latent Sampling:** Sample $\mathbf{z}$ from $q_\phi(\mathbf{z}|\mathbf{I})$ using the reparameterization trick: $\mathbf{z} = \boldsymbol{\mu} + \boldsymbol{\epsilon} \odot \boldsymbol{\sigma}$, where $\boldsymbol{\epsilon} \sim \mathcal{N}(0, \mathbf{I})$ [S1][G].
3.  **Scene Representation:** The attention-based scene function $F_\Theta(\mathbf{x}, \mathbf{d}, \mathbf{z})$ predicts volume density $\sigma$ and view-dependent color $\mathbf{c}$ for any point $\mathbf{x}$ and direction $\mathbf{d}$ conditioned on $\mathbf{z}$ [S1].
4.  **Rendering:** Render the image $\hat{I}$ by integrating $\sigma$ and $\mathbf{c}$ along rays using standard volume rendering equations [S1][G].
5.  **Training (Iterative Amortized Inference):** Optimize encoder parameters $\phi$ and scene function parameters $\Theta$ by minimizing the VAE loss:
    $$ \mathcal{L} = \mathbb{E}_{q_\phi(\mathbf{z}|\mathbf{I})} [\log p_\Theta(\mathbf{I}|\mathbf{z})] - \beta \cdot \text{KL}(q_\phi(\mathbf{z}|\mathbf{I}) || p(\mathbf{z})) $$
    where $p(\mathbf{z}) = \mathcal{N}(0, \mathbf{I})$ is the prior, and $\beta$ is a weighting factor [S1][G]. The term $\log p_\Theta(\mathbf{I}|\mathbf{z})$ is approximated by the negative rendering loss (e.g., MSE between $\hat{I}$ and $I$) [S1][G].

**Parameters:**
-   **Latent Dimension:** Not given in the sources.
-   **$\beta$:** Not given in the sources.
-   **Encoder Architecture:** ResNet [S1].
-   **Scene Function:** Attention-based network [S1].

**Edge Cases and Pitfalls:**
-   **Low Expressivity:** The single latent code limits scene complexity, leading to lower fidelity than baseline NeRF when many views (100+) are available [S1].
-   **Ambiguity:** Sparse inputs (5-20 views) may lead to ambiguous latent codes, requiring robust amortized inference [S1].

## Subtopics

### [Iterative Amortized Inference](14565-optimization-and-training-procedures.md)
A training procedure that refines the latent code over multiple steps, improving inference accuracy for sparse views.

### [Attention-Based Scene Functions](11313-model-architecture-modifications.md)
Neural architectures using attention mechanisms to model spatial dependencies, used here to replace standard MLPs in NeRF-VAE.

## Operating characteristics
-   **Time Complexity:** Training time is significantly reduced compared to baseline NeRF as it avoids per-scene optimization; inference is fast once the latent code is encoded [S1].
-   **Memory Complexity:** Requires storing the encoder and scene function parameters; memory usage is independent of scene size but depends on latent dimension and network depth [S1].
-   **Performance:** Outperforms baseline NeRF with 5-20 input views but underperforms with 100+ views due to lower expressivity [S1].
-   **Strengths:** Generalizes to new scenes; no per-scene training required; works with sparse views.
-   **Limitations:** Lower fidelity for dense views; limited scene complexity due to single latent code.
-   **When to Use:** Sparse view synthesis, few-shot learning, or when per-scene optimization is prohibitive.
-   **When Not to Use:** High-fidelity reconstruction with many input views.

## Background
NeRF-VAE was proposed by Kosiorek et al. in January 2021 [S1]. It extends the NeRF framework by integrating it into a VAE structure, enabling generative modeling of 3D scenes [S1][S2, unofficial]. The method was evaluated on standard GAN datasets such as Celeb-A, CARLA, and Cats [S1]. It builds on Iterative Amortized Inference [S1] and ResNet encoders [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
