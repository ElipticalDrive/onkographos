# pi-GAN

Also called: Periodic Implicit Generative Adversarial Networks, SIREN-based Implicit GAN

## Overview
pi-GAN is a generative adversarial network that learns an unsupervised, 3D-aware implicit radiance field from 2D images by conditioning a SIREN network on a latent noise vector [S1]. It differs from standard NeRF training by treating the radiance field as a generative prior conditioned on a global shape/appearance code $z$, rather than optimizing a specific scene's coordinates directly, and by using a mapping network with Feature-wise Linear Modulation (FiLM) instead of simple concatenation [S1]. Unlike GRAF, which uses concatenation-based conditioning and patch-based discriminators, pi-GAN employs a StyleGAN-inspired mapping network and a global convolutional discriminator to avoid local overfitting inherent to periodic activations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A latent noise vector $\mathbf{z} \sim p_z$ (typically from a standard normal distribution) and a camera pose $\xi$ (defined by origin $\mathbf{o}$ and direction $\mathbf{d}$).
*   **Output:** A synthesized 2D image $I$ rendered from the camera pose $\xi$.
*   **Implicit Representation:** A 5D spatio-angular radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ conditioned on $\mathbf{z}$.

**Data Structures:**
1.  **Mapping Network ($M$):** A Multi-Layer Perceptron (MLP) with ReLU activations.
    *   Input: $\mathbf{z} \in \mathbb{R}^{d_z}$.
    *   Output: A set of frequency parameters $\{\gamma_i\}_{i=0}^{n-1}$ and phase shift parameters $\{\beta_i\}_{i=0}^{n-1}$ for the SIREN layers.
2.  **SIREN Backbone ($\Phi$):** A sequence of $n$ layers.
    *   Each layer $i$ has weights $\mathbf{W}_i \in \mathbb{R}^{N_i \times M_i}$ and biases $\mathbf{b}_i \in \mathbb{R}^{N_i}$.
    *   Activated by sine functions modulated by $\gamma_i$ and $\beta_i$.
3.  **Heads:**
    *   Density Head: Linear layer $\mathbf{W}_\sigma, \mathbf{b}_\sigma$.
    *   Color Head: Linear layer $\mathbf{W}_c, \mathbf{b}_c$ followed by activation $\phi_c$ (typically ReLU or Sigmoid).
4.  **Discriminator ($D$):** A standard convolutional neural network (CNN) taking 2D images as input.

**Numbered Steps:**

1.  **Latent Mapping:**
    The noise vector $\mathbf{z}$ is passed through the Mapping Network $M$ to produce the conditioning parameters for the SIREN layers:
    $$
    \{\gamma_i, \beta_i\}_{i=0}^{n-1} = M(\mathbf{z})
    $$
    [S1]

2.  **SIREN Evaluation (Feature Extraction):**
    For a 3D point $\mathbf{x} \in \mathbb{R}^3$, compute the SIREN backbone output $\Phi(\mathbf{x})$ by composing layers $i=0$ to $n-1$:
    $$
    \mathbf{x}_0 = \mathbf{x}
    $$
    $$
    \mathbf{x}_{i+1} = \phi_i(\mathbf{x}_i) = \sin(\gamma_i \cdot (\mathbf{W}_i \mathbf{x}_i + \mathbf{b}_i) + \beta_i)
    $$
    where $\cdot$ denotes element-wise multiplication. The final feature vector is $\Phi(\mathbf{x}) = \mathbf{x}_n$.
    [S1]

3.  **Radiance Field Prediction:**
    Compute density $\sigma$ and color $\mathbf{c}$ using the feature vector $\Phi(\mathbf{x})$ and viewing direction $\mathbf{d}$:
    $$
    \sigma(\mathbf{x}) = \mathbf{W}_\sigma \Phi(\mathbf{x}) + \mathbf{b}_\sigma
    $$
    $$
    \mathbf{c}(\mathbf{x}, \mathbf{d}) = \mathbf{W}_c \phi_c \left( [\Phi(\mathbf{x}), \mathbf{d}]^T \right) + \mathbf{b}_c
    $$
    where $[\cdot, \cdot]^T$ denotes concatenation. $\phi_c$ is the activation function for the color head.
    [S1]

4.  **Neural Volume Rendering:**
    Cast a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ from camera origin $\mathbf{o}$ with direction $\mathbf{d}$. Sample $N$ points $t_i$ within bounds $[t_n, t_f]$. Let $\mathbf{x}_i = \mathbf{r}(t_i)$ and $\delta_i = t_{i+1} - t_i$.
    Compute transmittance $T_i$ and alpha $\alpha_i$:
    $$
    \alpha_i = 1 - \exp(-\sigma(\mathbf{x}_i) \delta_i)
    $$
    $$
    T_i = \prod_{j=1}^{i-1} (1 - \alpha_j), \quad T_1 = 1
    $$
    Compute pixel color $C(\mathbf{r})$:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}(\mathbf{x}_i, \mathbf{d})
    $$
    [S1, Parent]

5.  **Adversarial Training:**
    *   **Generator Loss:** The rendered image $I_{gen}$ is passed to the Discriminator $D$. The generator minimizes the adversarial loss (e.g., non-saturating GAN loss) and potentially a reconstruction loss if supervised, though pi-GAN is primarily unsupervised.
    *   **Discriminator Loss:** The discriminator distinguishes between real images $I_{real}$ and generated images $I_{gen}$.
    *   **Progressive Growing:** Training starts at low resolution and progressively adds layers to the generator and discriminator to stabilize training and accelerate convergence [S1].

**Parameters:**
*   $\mathbf{z}$: Latent vector dimension (e.g., 512).
*   $n$: Number of SIREN layers.
*   $\gamma_i, \beta_i$: Learned frequencies and phase shifts per layer.
*   $t_n, t_f$: Near and far clipping planes for ray sampling.
*   $N$: Number of samples per ray.

**Edge Cases and Pitfalls:**
*   **Local Overfitting:** SIREN networks are prone to local overfitting to the last batch if sufficient spatial coverage is not maintained during training. This is why pi-GAN avoids patch-based discriminators (used in GRAF) and uses a global convolutional discriminator instead [S1].
*   **Periodic Ambiguity:** The sine activation introduces periodicity, which can lead to high-frequency artifacts if frequencies $\gamma_i$ are not properly scaled or learned. The mapping network learns these frequencies, but initialization matters.
*   **Density Positivity:** While $\sigma$ is output directly, it should be non-negative for physical validity. The sources do not explicitly state a clamp or softplus is applied to $\sigma$ in the final formula, but standard volume rendering assumes $\sigma \ge 0$. [G: In practice, $\sigma$ is often clamped to $\ge 0$ or passed through a softplus to ensure valid transmittance calculations, though the source formula (3) shows a linear output. Implementers should verify if a clamp is required for stability.]

## Subtopics

### SIREN (Sinusoidal Representation Networks)
The backbone architecture using sine activations. It allows the network to represent high-frequency functions and derivatives, crucial for capturing sharp details in radiance fields. It differs from ReLU-based MLPs (like standard NeRF) by avoiding spectral bias towards low frequencies. [Link to SIREN if available, otherwise note as component].

### FiLM (Feature-wise Linear Modulation)
A conditioning mechanism where parameters $\gamma$ and $\beta$ scale and shift the activations. It is more expressive than concatenation because it allows the network to modulate its internal representations dynamically based on the latent code. [Link to FiLM if available].

### Progressive Growing
A training strategy where the network resolution is increased gradually. This stabilizes GAN training and reduces computational cost during early stages. [Link to Progressive Growing if available].

## Operating characteristics
*   **Time Complexity:** Training is computationally expensive due to the iterative evaluation of the SIREN network for every sample along every ray during every iteration. The use of progressive growing mitigates this by starting with fewer samples and lower resolution.
*   **Memory Complexity:** The SIREN weights are stored in memory. The mapping network is small. The discriminator is a standard CNN. No large voxel grids or hash tables are stored, making it memory-efficient compared to voxel-based methods, but the MLP evaluation is slower than texture lookups.
*   **Strengths:**
    *   **3D Awareness:** Produces view-consistent images from arbitrary camera poses.
    *   **High Quality:** Achieves state-of-the-art results on datasets like CelebA, Cats, and CARLA [S1].
    *   **Interpretability:** The implicit density field allows extraction of 3D proxy shapes using marching cubes [S1].
    *   **Unsupervised:** Learns 3D structure from unlabeled 2D images.
*   **Limitations:**
    *   **Training Stability:** GANs are notoriously unstable. The specific choice of discriminator and conditioning is critical.
    *   **Local Overfitting Risk:** As noted, requires careful handling of batch coverage and discriminator design to prevent SIREN from overfitting to local regions.
    *   **No Explicit Geometry:** Unlike methods using signed distance functions (SDFs), pi-GAN uses density, which may result in less precise geometric boundaries for thin structures.
*   **When to Use:** When 3D-aware image synthesis from unlabeled 2D data is required, and high visual fidelity is prioritized over explicit geometric precision or training speed.
*   **When Not to Use:** When explicit, watertight geometry is needed (consider SDF-based methods), or when training data is limited and supervised signals (like depth) are available (consider supervised NeRF variants).

## Background
pi-GAN was introduced by Tewari et al. in 2021 [S1]. It builds upon the success of StyleGAN in 2D image synthesis and the recent emergence of neural radiance fields (NeRF) for 3D representation. The key innovation was adapting the StyleGAN architecture (mapping network + style-based generator) to implicit 3D representations using SIREN networks. Previous attempts like GRAF used concatenation-based conditioning and patch-based discriminators, which pi-GAN found to be less effective due to SIREN's sensitivity to local overfitting. The method demonstrated that periodic activations combined with FiLM conditioning could effectively model complex 3D shapes and textures in an unsupervised manner.

## Sources
[S1] pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis, https://doi.org/10.1109/cvpr46437.2021.00574, fetched 2026-10-26
