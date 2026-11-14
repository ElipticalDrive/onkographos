# NeRF in the Wild (NeRF-W)
Also called: NeRF-W

## Overview
NeRF in the Wild (NeRF-W) addresses the failure of standard NeRF models when applied to real-world image collections containing per-image appearance variations (due to lighting changes) and transient objects (such as moving people or vehicles) [S1]. It sets itself apart from standard NeRF by introducing per-image appearance embeddings to condition the color prediction and a separate transient network to model occluders that do not belong to the static scene geometry [S1]. Unlike [Spherical Harmonics NeRF](14480-spherical-harmonics-nerf.md), which models lighting analytically, NeRF-W learns appearance variations implicitly through embeddings [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of images $\{I_i\}$ with known camera poses, scene coordinates $\mathbf{x}$, view directions $\mathbf{d}$, and an image index $i$.
*   **Outputs:** A rendered color $\hat{C}_i(\mathbf{r})$ for a ray $\mathbf{r}$ originating from camera $i$.

**Data Structures:**
1.  **Static Density MLP ($\mathcal{M}_\sigma$):** Takes positional encoded position $\gamma(\mathbf{x})$. Outputs static volume density $\sigma$. This network is shared across all images and is **not** conditioned on the image index [S1].
2.  **Static Color MLP ($\mathcal{M}_c$):** Takes the latent feature vector $\mathbf{h}$ (output of $\mathcal{M}_\sigma$), positional encoded position $\gamma(\mathbf{x})$, positional encoded view direction $\gamma(\mathbf{d})$, and a **per-image appearance embedding** $\mathbf{z}_i$. Outputs static color $\mathbf{c}$ [S1].
3.  **Transient MLP ($\mathcal{M}_\tau$):** Takes positional encoded position $\gamma(\mathbf{x})$, positional encoded view direction $\gamma(\mathbf{d})$, and a **per-image transient embedding** $\mathbf{t}_i$. Outputs transient color $\mathbf{c}^{(\tau)}$ and transient density $\sigma^{(\tau)}$ [S1].

**Parameters:**
*   $\mathbf{z}_i$: Appearance embedding vector for image $i$.
*   $\mathbf{t}_i$: Transient embedding vector for image $i$.
*   $N$: Number of sample points along a ray.

**Steps:**
1.  **Sample Points:** For each ray $\mathbf{r}$ from camera $i$, sample $N$ points $\mathbf{x}_n$ along the ray.
2.  **Encode Inputs:** Compute positional encodings $\gamma(\mathbf{x}_n)$ and $\gamma(\mathbf{d})$.
3.  **Predict Static Properties:**
    *   Compute $\mathbf{h}_n = \mathcal{M}_\sigma(\gamma(\mathbf{x}_n))$.
    *   Compute static density $\sigma_n = \mathcal{M}_\sigma^{out}(\mathbf{h}_n)$ (or directly from $\mathcal{M}_\sigma$ if structured as single output).
    *   Compute static color $\mathbf{c}_n = \mathcal{M}_c(\mathbf{h}_n, \gamma(\mathbf{x}_n), \gamma(\mathbf{d}), \mathbf{z}_i)$.
4.  **Predict Transient Properties:**
    *   Compute transient density $\sigma_n^{(\tau)} = \mathcal{M}_\tau^{\sigma}(\gamma(\mathbf{x}_n), \gamma(\mathbf{d}), \mathbf{t}_i)$.
    *   Compute transient color $\mathbf{c}_n^{(\tau)} = \mathcal{M}_\tau^{c}(\gamma(\mathbf{x}_n), \gamma(\mathbf{d}), \mathbf{t}_i)$.
5.  **Composite:** Combine static and transient contributions using standard volume rendering integration [S1].

**Formulas:**
The total density at sample $n$ is:
$$ \hat{\sigma}_n = \sigma_n + \sigma_n^{(\tau)} $$

The opacity at sample $n$ is:
$$ \alpha_n = 1 - \exp(-\hat{\sigma}_n \delta_n) $$
where $\delta_n$ is the distance between samples.

The transmittance at sample $n$ is:
$$ T_n = \exp\left(-\sum_{j=1}^{n-1} \hat{\sigma}_j \delta_j\right) $$

The rendered color is the sum of static and transient contributions:
$$ \hat{C}_i(\mathbf{r}) = \sum_{n=1}^{N} T_n \alpha_n (\mathbf{c}_n + \mathbf{c}_n^{(\tau)}) $$

**Edge Cases and Pitfalls:**
*   **Transient Embedding Initialization:** The transient embeddings $\mathbf{t}_i$ must be initialized carefully to avoid all pixels being classified as transient or static initially [G].
*   **Identity Ambiguity:** If the transient network is too powerful, it may absorb static scene details, leading to incorrect pose estimation or rendering artifacts [S2, unofficial].

## Subtopics

### Transient Filtering via Candidate Head
An architectural variant that uses a candidate head to predict transient properties alongside static properties, enabling robust pose optimization by filtering out transient occluders [S2, unofficial]. It differs from NeRF-W by isolating the transient network and using deep features (DINO) for candidate filtering, whereas NeRF-W does not isolate the transient network and relies on different architecture assumptions [S2, unofficial].
*   **Definition:** Uses a candidate head to predict transient color $\mathbf{c}_i^{(\tau)}$, transient density $\sigma_i^{(\tau)}$, and beta $\beta_i$ using predicted candidate feature $\mathbf{f}_i$ and candidate embedding $\boldsymbol{\ell}_i^{(c)}$ [S2, unofficial].
*   **Step:** Render pixel color $\hat{C}_i(\mathbf{r})$ using static and transient parameters ($\mathbf{c}_i, \mathbf{c}_i^{(\tau)}, \sigma_i, \sigma_i^{(\tau)}$) following the NeRF-W compositing scheme [S2, unofficial].
*   **Pitfall:** Merging transient and candidate heads degrades performance; separating the transient head is necessary, as failing to do so causes the model to fail in estimating correct poses and reduces rendering quality [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time increases due to the additional MLPs and embeddings. Inference time is slightly higher due to the transient network evaluation [S1].
*   **Memory Complexity:** Memory usage increases by the size of the appearance and transient embeddings for each image in the dataset [S1].
*   **Strengths:** Handles dynamic scenes and varying lighting conditions effectively [S1].
*   **Limitations:** Requires careful tuning of transient vs. static separation to avoid artifacts [S2, unofficial]. Performance may degrade if transient objects are not properly identified [S2, unofficial].
*   **When to use:** Real-world image collections with significant lighting changes and transient objects [S1].
*   **When not to use:** Controlled studio environments with static scenes and consistent lighting [S1].

## Background
NeRF in the Wild (NeRF-W) was introduced to address the limitations of early NeRF models when applied to real-life photographs [S1]. It specifically targets issues related to per-image appearance variations due to lighting conditions and the presence of transient objects that differ in each image [S1]. The method keeps the density MLP fixed for all images in a scene while conditioning the color MLP on per-image appearance embeddings [S1]. Another MLP conditioned on per-image transient embeddings predicts the color and density functions of transient objects [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
