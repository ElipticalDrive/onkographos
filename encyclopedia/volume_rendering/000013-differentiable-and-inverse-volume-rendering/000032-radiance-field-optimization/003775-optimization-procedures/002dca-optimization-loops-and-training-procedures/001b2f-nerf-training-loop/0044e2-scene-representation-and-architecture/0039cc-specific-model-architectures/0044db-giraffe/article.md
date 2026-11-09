# GIRAFFE

Also called: Compositional Generative Neural Feature Fields, GIRAFFE Scene Composition

## Overview
GIRAFFE represents scenes as a composition of $N$ distinct entities (objects plus a background), where each entity possesses its own neural feature field and an affine transformation. This architecture enables explicit control over the pose, shape, and appearance of individual objects within a scene [S1]. It differs from monolithic NeRF representations by disentangling scene elements into separate learnable fields, allowing for generative manipulation of specific entities rather than just the global scene [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera pose $\xi$, number of entities $N$, shape latent codes $\mathbf{z}_s^i$, appearance latent codes $\mathbf{z}_a^i$, and transformation parameters $\mathbf{T}_i = \{\mathbf{s}_i, \mathbf{t}_i, \mathbf{R}_i\}$ for each entity $i \in \{1, \dots, N\}$.
*   **Outputs:** A synthesized RGB image $\hat{\mathbf{I}} \in \mathbb{R}^{H \times W \times 3}$.

**Data Structures:**
1.  **Feature Field $h_\theta$:** An MLP shared by all object entities (index $1$ to $N-1$).
    *   **Architecture:** 8 layers, hidden dimension 128, ReLU activations.
    *   **Input:** Concatenated positional encoding $\gamma(k^{-1}(\mathbf{x}))$ and directional encoding $\gamma(k^{-1}(\mathbf{d}))$, plus latents $\mathbf{z}_s, \mathbf{z}_a$.
    *   **Output:** Density $\sigma \in \mathbb{R}$ and feature vector $\mathbf{f} \in \mathbb{R}^{M_f}$.
    *   **Background Field:** The background entity ($i=N$) uses a separate field with half the layers and hidden dimension of the object field [S1].
2.  **Transformation Parameters $\mathbf{T}_i$:**
    *   $\mathbf{s}_i \in \mathbb{R}^3$: Scale.
    *   $\mathbf{t}_i \in \mathbb{R}^3$: Translation.
    *   $\mathbf{R}_i \in SO(3)$: Rotation matrix.
    *   **Constraint:** For the background entity $N$, $\mathbf{s}_N$ and $\mathbf{t}_N$ are fixed to span the entire scene and center at the origin. Other entities have learnable $\mathbf{s}, \mathbf{t}, \mathbf{R}$ [S1].
3.  **Feature Image $I_V$:** Low-resolution feature map of size $H_V \times W_V \times M_f$ (typically $16 \times 16$) [S1].
4.  **Neural Renderer $\pi_\theta^{\text{neural}}$:** A 2D CNN mapping $I_V$ to RGB.

**Numbered Steps:**

1.  **Sample Entities and Parameters:**
    *   Sample number of entities $N \sim p_N$.
    *   Sample latent codes $\mathbf{z}_s^i, \mathbf{z}_a^i \sim \mathcal{N}(0, I)$ for each entity $i$.
    *   Sample camera pose $\xi \sim p_\xi$ (uniform over dataset-dependent elevation angles) [S1].
    *   Sample transformations $\mathbf{T}_i \sim p_T$ (uniform over valid object transformations) for $i=1 \dots N-1$. For $i=N$, use fixed background transforms [S1].

2.  **Define Transformation Function:**
    *   For each entity $i$, define the mapping from object space to scene space $k_i(\mathbf{x})$:
        $$k_i(\mathbf{x}) = \mathbf{R}_i \begin{bmatrix} s_{i,1} & 0 & 0 & 0 \\ 0 & s_{i,2} & 0 & 0 \\ 0 & 0 & s_{i,3} & 0 \\ 0 & 0 & 0 & s_{i,3} \end{bmatrix} \mathbf{x} + \mathbf{t}_i$$
        Note: The matrix uses $s_{i,3}$ for the homogeneous coordinate scaling in the provided source notation [S1].

3.  **Volume Rendering Loop (for each pixel ray $\mathbf{d}_k$):**
    *   Sample $N_s = 64$ points $\mathbf{x}_j$ along the ray $\mathbf{d}_k$ for $j=1 \dots N_s$ [S1].
    *   For each sample point $\mathbf{x}_j$:
        a.  **Transform to Object Space:** Compute inverse transformation for each entity $i$:
            $$(\mathbf{x}_{obj}, \mathbf{d}_{obj}) = (k_i^{-1}(\mathbf{x}_j), k_i^{-1}(\mathbf{d}_k))$$
            [S1]
        b.  **Encode Inputs:** Apply positional encoding $\gamma$ to $\mathbf{x}_{obj}$ and $\mathbf{d}_{obj}$.
            *   Positional encoding dimensions: $L_x = 2 \cdot 3 \cdot 10$ for position, $L_d = 2 \cdot 3 \cdot 4$ for direction [S1].
        c.  **Query Feature Field:** Pass encoded inputs and latents $\mathbf{z}_s^i, \mathbf{z}_a^i$ to $h_\theta$ to get $(\sigma_{i,j}, \mathbf{f}_{i,j})$.
        d.  **Compose Entities:** Combine densities and features from all $N$ entities using the composition operator $C$:
            $$\sigma_j = \sum_{i=1}^N \sigma_{i,j}$$
            $$\mathbf{f}_j = \frac{1}{\sigma_j} \sum_{i=1}^N \sigma_{i,j} \mathbf{f}_{i,j}$$
            If $\sigma_j = 0$, $\mathbf{f}_j = \mathbf{0}$ [S1].
    *   **Integrate Features:** Compute the final feature vector $\mathbf{f}_k$ for the pixel using alpha compositing:
        $$\alpha_j = 1 - e^{-\sigma_j \delta_j}$$
        $$\tau_j = \prod_{k=1}^{j-1} (1 - \alpha_k)$$
        $$\mathbf{f}_k = \sum_{j=1}^{N_s} \tau_j \alpha_j \mathbf{f}_j$$
        where $\delta_j = ||\mathbf{x}_{j+1} - \mathbf{x}_j||_2$ [S1].

4.  **Construct Feature Image:**
    *   Assemble $\mathbf{f}_k$ for all pixels into the low-resolution feature image $I_V \in \mathbb{R}^{16 \times 16 \times M_f}$ [S1].

5.  **2D Neural Rendering:**
    *   Upsample $I_V$ to high-resolution RGB image $\hat{\mathbf{I}} \in \mathbb{R}^{H \times W \times 3}$ using $\pi_\theta^{\text{neural}}$.
    *   **Architecture:** 2D CNN with leaky ReLU. Combines nearest neighbor upsampling with $3 \times 3$ convolutions. Maps feature image to RGB at every spatial resolution via $3 \times 3$ conv, adding to previous output via bilinear upsampling (skip connections). Applies sigmoid to final RGB layer [S1].

**Parameters:**
*   $N_s = 64$: Number of sample points per ray [S1].
*   $M_f$: Feature dimension. 128 for 64x64 output, 256 for 256x256 output [S1].
*   Generator Learning Rate: $1 \times 10^{-4}$ (or $2.5 \times 10^{-4}$ for 256x256) [S1].
*   Discriminator Learning Rate: $5 \times 10^{-4}$ [S1].
*   Batch Size: 32 [S1].
*   EMA Decay: 0.999 [S1].
*   GAN Penalty $\lambda$: 10 [S1].

**Edge Cases and Pitfalls:**
*   **Background Handling:** The background entity must have fixed scale and translation to cover the scene; failing to fix these allows the background to compete incorrectly with objects [S1].
*   **Density Summation:** The composition operator sums densities. If multiple objects occupy the same space, their densities add up, ensuring gradient flow to all entities with density $>0$ [S1].
*   **Resolution Mismatch:** Rendering features at 16x16 and upsampling is a design choice for speed and quality; rendering directly at high resolution is not supported by this specific pipeline [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is performed at low resolution (16x16) for features, reducing the cost of the 3D volume rendering step. The 2D neural renderer handles the upscaling, which is generally faster than high-resolution 3D sampling [S1].
*   **Memory Complexity:** Shared weights for object feature fields reduce memory usage compared to storing separate networks for each object instance. The background field uses half the capacity [S1].
*   **Strengths:** Explicit disentanglement of objects allows for generative control of individual entities (pose, shape, appearance). The composition operator ensures gradients flow to all visible objects [S1].
*   **Limitations:** Assumes a limited number of entities ($N$). Performance depends on the quality of the 2D upsampling network. Background handling requires specific constraints [S1].
*   **When to use:** When scene disentanglement and generative control of individual objects are required. Suitable for datasets with limited background complexity (e.g., white background) [S1].
*   **When not to use:** For scenes with unbounded or highly complex backgrounds where a fixed background entity is insufficient.

## Background
GIRAFFE was introduced in the paper "GIRAFFE: Representing Scenes as Compositional Generative Neural Feature Fields" [S1]. It builds on the concept of neural radiance fields but extends them to a generative setting with explicit scene composition. The method is designed for single-object datasets like Chairs, Cats, and CelebA, where the background is simple [S1].

## Sources
[S1] GIRAFFE: Representing Scenes as Compositional Generative Neural Feature Fields, https://github.com/chaerinkong/paper-reviews/issues/63, fetched 2026-10-28
