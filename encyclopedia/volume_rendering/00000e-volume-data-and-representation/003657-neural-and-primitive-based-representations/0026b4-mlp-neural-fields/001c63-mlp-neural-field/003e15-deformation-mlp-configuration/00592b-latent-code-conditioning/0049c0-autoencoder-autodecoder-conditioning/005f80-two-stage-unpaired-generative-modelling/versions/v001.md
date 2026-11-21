# Two-Stage Unpaired Generative Modelling

Also called: 3D-aware GAN with CPDDF, CPDDF-based Generative Modelling (VAE + GAN), Shape VAE with CPDDF, Unpaired 2D-3D Generative Modelling

## Overview
This method is a two-stage architecture for 3D-to-2D translation that generates photorealistic images from unpaired 3D shape data and 2D image data [S2]. It decouples geometry learning from texture synthesis: Stage 1 trains a Variational Autoencoder (VAE) to map 3D point clouds to a latent shape vector using a Conditional Probabilistic Directed Distance Field (CPDDF) [S1]. Stage 2 employs a Generative Adversarial Network (GAN) to translate rendered surface normals (derived from the latent shape) into RGB images, conditioned on a separate latent texture vector [S1]. It is distinguished from paired translation methods by its ability to train on unpaired datasets, and from pure 2D GANs by its explicit 3D-aware geometric representation [S2].

## How it works
**Inputs and Outputs:**
*   **Stage 1 Input:** 3D point clouds $P$ with associated normals $N$.
*   **Stage 1 Output:** Latent shape vector $z_s$ and a learned CPDDF decoder.
*   **Stage 2 Input:** Rendered surface normal image $I_n$, sampled latent texture vector $z_T$, and camera parameters $\Pi$.
*   **Stage 2 Output:** Generated RGB image $I_T$.

**Data Structures:**
1.  **Point Cloud:** A set of $K$ points $P = \{p_i\}$ with normals $N = \{n_i\}$. Typical size: 1024 points per shape for the encoder [S1].
2.  **Latent Shape Vector ($z_s$):** A vector sampled from a Gaussian distribution, output by the PointNet encoder [S1].
3.  **Latent Texture Vector ($z_T$):** A vector sampled from a distribution, used to condition the texture generator [S1].
4.  **CPDDF:** A neural field parameterized by $z_s$ that outputs depth values $d(p, v|z_s)$ for a ray origin $p$ and direction $v$ [S1].

**Numbered Steps:**

1.  **Stage 1: Shape VAE Training**
    1.  Sample a 3D shape from the dataset (e.g., ShapeNet cars) [S1].
    2.  Encode the point cloud $(P, N)$ into a latent shape vector $z_s$ using a PointNet encoder $E$:
        $$z_s = E(P, N)$$
        [S1]
    3.  Decode $z_s$ using a CPDDF. The CPDDF computes depth values $d(p, v|z_s)$ for rays. Implementation uses a modulated SIREN approach to handle the conditioning [S1].
    4.  Compute the VAE loss $\mathcal{E}_{\text{VAE}}$:
        $$\mathcal{E}_{\text{VAE}} = \mathcal{E}_S + \beta \mathcal{L}_{\text{KL}}$$
        where $\mathcal{L}_{\text{KL}}$ is the KL divergence between the approximate posterior and a Gaussian prior, and $\mathcal{E}_S$ is the reconstruction likelihood term [S1].
    5.  Optimize the encoder $E$ and CPDDF decoder parameters to minimize $\mathcal{E}_{\text{VAE}}$ [S1].

2.  **Stage 2: GAN Translation**
    1.  Sample a latent shape vector $z_s$ (from the trained VAE or distribution) and camera parameters $\Pi$ [S1].
    2.  Render a surface normal image $I_n$ by computing the derivatives of the learned CPDDF defined by $z_s$ with respect to the camera view $\Pi$ [S1].
    3.  Sample a latent texture vector $z_T$ [S1].
    4.  Concatenate the normal map $I_n$ with $z_T$:
        $$I_{\text{cond}} = I_n \oplus z_T$$
        [S1]
    5.  Generate the final RGB image $I_T$ using a texture generator network $f_T$:
        $$I_T = f_T(I_n, z_T)$$
        [S1]
    6.  Train the GAN using cycle-consistent image-to-image translation losses to ensure the generated RGB images match the distribution of real images [S2].

**Parameters:**
*   $\beta$: Weight for the KL divergence term in the VAE loss [S1].
*   Training Data: Single categories of ShapeNet (e.g., cars) [S1].
*   Point Cloud Sampling: 1024 points sent to the encoder; 1.2K (A/U) or 300 (S/B/T/O) oriented points per shape for training batches [S1].
*   Batch Size: 32 shapes per minibatch [S1].

**Edge Cases and Pitfalls:**
*   **Atypical Inputs:** The network may fail to produce high-quality geometry for atypical inputs or categories with insufficient training examples (e.g., specific chair legs or car wheels) [S1].
*   **Sampling Density:** Non-uniform densities from the point cloud sampling algorithm can lead to concentrations of points on specific features (e.g., chair legs, wheels), potentially affecting reconstruction quality [S1].
*   **Unpaired Data Limitation:** While it avoids paired data requirements, it underperforms compared to a 2D image GAN trained with the same critic architecture [S2].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Outperforms samples from image VAEs and GAN-based textured low-poly 3D mesh renders in image quality [S2]. Underperforms a 2D image GAN with the same critic [S2].
*   **Strengths:** Achieves 3D-aware generation using unpaired 2D-3D data; leverages versatile CPDDF representations [S2].
*   **Limitations:** Quality degrades on atypical shapes or categories with sparse data; lower quality than specialized 2D GANs [S1, S2].
*   **When to use:** When paired 3D-2D data is unavailable, but 3D shape data (point clouds) and 2D images are available for separate training [S2].

## Background
The method was introduced in the context of Probabilistic Directed Distance Fields (PDDF) for 3D shape representation [S1]. It extends the use of CPDDFs from deterministic shape representation to generative modelling, specifically addressing the challenge of 3D-aware image generation without paired 3D-2D data [S2]. The approach combines a Shape VAE (using PointNet and CPDDF) with a cycle-consistent GAN for texture synthesis [S1, S2].

## Sources
[S1] Representing 3D Shapes with Probabilistic Directed Distance Fields, https://doi.org/10.1109/cvpr52688.2022.01874, fetched 2026-10-26
[S2] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
