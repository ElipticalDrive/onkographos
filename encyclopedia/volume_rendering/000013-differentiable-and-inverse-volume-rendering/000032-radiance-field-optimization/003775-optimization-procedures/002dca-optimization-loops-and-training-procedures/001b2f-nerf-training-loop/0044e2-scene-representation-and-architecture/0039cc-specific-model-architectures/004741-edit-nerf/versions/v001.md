# Edit-NeRF


## Overview
Edit-NeRF is a method for user-controlled scene editing in Neural Radiance Fields (NeRF) by introducing latent codes for shape and appearance, allowing the model to be deformed based on 2D image conditioning [S1, S2]. It differs from standard NeRF by formulating editing as a joint optimization problem of network parameters and latent codes, and differs from later text-driven methods like CLIP-NeRF by relying on direct image-based latent conditioning rather than semantic text embeddings [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images of objects from a specific category; user-provided 2D image conditioning for editing (shape/appearance guidance).
*   **Outputs:** A rendered image $C(\mathbf{r}, z_s, z_c)$ and the optimized NeRF representation (network weights and latent codes).

**Data Structures:**
The model uses a modular conditional radiance field architecture [S2].
1.  **Shared Shape Network ($F_{share}$):** A category-specific network that is not conditioned on instance-specific latent codes. It acts as a deformation field [S1].
2.  **Instance-Specific Shape Network ($F_{inst}$):** Conditioned on a shape latent code $z_s$. It generates instance-specific shape variations [S1].
3.  **Latent Codes:**
    *   $z_s$: Shape latent code.
    *   $z_a$ (or $z_c$): Appearance latent code [S1, S2].
4.  **Network Weights:** The parameters of the MLPs comprising $F_{share}$, $F_{inst}$, and the color prediction network.

**Numbered Steps:**
1.  **Training Phase:** Train the model on multiple instances of the same object category. The model learns underlying part features in an unsupervised manner [S2].
2.  **Optimization Loop (GLO):** For a given editing task, perform a Joint Latent Optimization (GLO) process [S1]:
    1.  Optimize the latent codes ($z_s, z_a$) while keeping MLP weights fixed, minimizing the NeRF photometric loss.
    2.  Optimize the MLP weights while keeping latent codes fixed.
    3.  Optimize both latent codes and MLP weights jointly [S1].
3.  **Rendering:** Compute the pixel color for a ray $\mathbf{r}$ using the volumetric rendering equation with the predicted density $\sigma_i$ and color $\mathbf{c}_i$ [S2]:
    $$
    \hat{C}(\mathbf{r}, z_s, z_c) = \sum_{i=1}^{N} c_i \alpha_i \exp \left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right)
    $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, $\delta_i$ is the distance between samples, and $N$ is the number of samples [S2].

**Formulas:**
*   **Volumetric Rendering:**
    $$
    \hat{C}(\mathbf{r}, z_s, z_c) = \sum_{i=1}^{N} c_i \alpha_i T_i, \quad T_i = \exp \left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right)
    $$
    $$
    \alpha_i = 1 - \exp(-\sigma_i \delta_i)
    $$
    [S2]

**Parameters:**
*   $z_s, z_a$: Latent vectors. Dimensions not given in the sources.
*   $N$: Number of samples per ray. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Limited Shape Operations:** The method only supports adding or deleting local parts of an object; it lacks support for complex global deformations [S2].
*   **Speed:** The editing process is slow due to the iterative optimization nature (GLO) [S2].
*   **Initialization:** The optimization requires careful initialization of latent codes and weights to converge, as it involves joint optimization of discrete latent codes and continuous network weights [S1].

## Operating characteristics
*   **Time Complexity:** High. The joint optimization of latent codes and network weights (GLO) is iterative and computationally expensive compared to feed-forward inference [S2].
*   **Memory Complexity:** Requires storing latent codes $z_s, z_a$ for each instance/editing session, in addition to the NeRF weights [S1].
*   **Strengths:** First method to enable user-controlled deformation on NeRF; allows propagation of 2D image guidance to 3D space [S2].
*   **Limitations:** Limited to local part editing (add/delete); slow convergence; requires multiple training instances of the same category for unsupervised part feature learning [S2].
*   **When to use:** When user-controlled, image-based editing of specific object parts is required and computational time is not a primary constraint.
*   **When not to use:** For real-time editing, global shape deformation, or when only text-based editing is desired (use CLIP-NeRF instead) [S2].

## Background
Edit-NeRF was proposed by Liu et al. in June 2021 [S1, S2]. It is recognized as the first method to execute user-controlled deformation models on NeRF, proposing a modular network's conditional radiance field to guide model editing [S2]. It builds on the standard latent-conditioned NeRF architecture but explicitly formulates the editing task as a joint optimization problem [S1]. It was succeeded by CLIP-NeRF (December 2021), which introduced text-based editing and greater freedom in shape operations [S1, S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
