# ReconFusion

Also called: ReconFusion: 3D Reconstruction with Diffusion Priors

## Overview
ReconFusion is a 3D reconstruction method that recovers high-quality scenes from a small number of input images (few-view) by regularizing a NeRF-based optimization pipeline with a diffusion model prior trained for novel view synthesis [S1]. Unlike methods that rely solely on reconstruction losses or text-guided score distillation, ReconFusion uses a multiview-conditioned diffusion model to generate plausible geometry and texture for unobserved regions, preventing catastrophic failures in underconstrained scenarios and reducing "floater" artifacts in denser captures [S1]. It differs from SparseFusion by sampling a new image at each iteration rather than using a multistep sampling approach, and empirically outperforms standard Score Distillation Sampling (SDS) [S1].

## How it works
ReconFusion consists of two main components: a diffusion model trained for novel view synthesis and a 3D reconstruction procedure that optimizes a NeRF using this diffusion prior [S1].

### Inputs and Outputs
**Inputs:**
- A set of $N$ posed input images $x_{obs} = \{x_i\}_{i=1}^N$ with corresponding camera parameters $\pi_{obs} = \{\pi_i\}_{i=1}^N$ [S1].
- A target novel camera pose $\pi$ for which a sample is required [S1].
- NeRF parameters $\psi$ (optimized) [S1].

**Outputs:**
- Optimized NeRF parameters $\psi$ representing the 3D scene [S1].

### Data Structures
1.  **NeRF Backbone:** Uses Zip-NeRF [2] as the backbone [S1].
2.  **Diffusion Model:** A Latent Diffusion Model (LDM) [42] with the following structure:
    -   **VAE Encoder ($E$):** Encodes input images to a latent representation.
    -   **Denoising U-Net ($\epsilon_\theta$):** Performs diffusion on latents.
    -   **VAE Decoder ($D$):** Recovers images from latents.
    -   **Input Resolution:** $512 \times 512 \times 3$ [S1].
    -   **Latent Space Dimensions:** $64 \times 64 \times 8$ [S1].
    -   **Conditioning:** The U-Net is augmented with two new conditioning pathways:
        -   **PixelNeRF Feature Map:** A feature map $f$ rendered by a PixelNeRF-style model $R_\phi$ for the target viewpoint $\pi$, using input images $x_{obs}$ and poses $\pi_{obs}$ [S1].
            -   $f = R_\phi(x_{obs}, \pi_{obs}, \pi)$ [S1].
            -   **Feature Map Resolution:** $64 \times 64$ with 128 channels [S1].
        -   **CLIP Embeddings:** Cross-attention mechanism using CLIP embeddings of the input images [S1].
3.  **PixelNeRF Encoder:** A small U-Net that takes an image of resolution $512 \times 512$ and outputs the feature map $f$ [S1].

### Parameters
-   **Diffusion Model Training:**
    -   **Datasets:** Finetuned from a pretrained LDM on a mixture of RealEstate10K [66], CO3D [38], MVImgNet [64], and Objaverse [10] [S1].
    -   **Batch Size:** 256 [S1].
    -   **Learning Rate:** $10^{-4}$ [S1].
    -   **Iterations:** 250k iterations for joint training of PixelNeRF and finetuning of the denoising U-Net [S1].
    -   **Classifier-Free Guidance (CFG):** Input images are set to all zeros randomly with probability 10% [S1].
-   **NeRF Optimization:**
    -   **Backbone:** Zip-NeRF [2].
    -   **Iterations:** 1000 iterations [S1].
    -   **Reconstruction Loss:** Charbonnier loss [8] [S1].
    -   **Sampling Loss Weight ($L_{sample}$):** Linearly decayed from 1 to 0.1 over training [S1].
    -   **CFG Scale:** 3.0 [S1].
    -   **Time Steps:**
        -   $t_{max}$: Fixed at 1.0 for all training steps [S1].
        -   $t_{min}$: Linearly annealed from 1.0 to 0.0 during training [S1].
    -   **Denoising Steps ($k$):** Fixed at 10 steps for DDIM sampling [S1].
-   **View Selection:** For a target novel view, the 3 nearest camera positions from the observed inputs are selected to condition the model [S1].

### Algorithm Steps

1.  **Diffusion Model Training (Preparation):**
    1.  Train a PixelNeRF-style model $R_\phi$ to render feature maps $f$ from input images $x_{obs}$ and poses $\pi_{obs}$ for a target pose $\pi$ [S1].
    2.  Finetune a pretrained LDM on multiview datasets (RealEstate10K, CO3D, MVImgNet, Objaverse) [S1].
    3.  Augment the LDM's U-Net with PixelNeRF feature map conditioning and CLIP cross-attention conditioning [S1].
    4.  Enable classifier-free guidance by zeroing input images with 10% probability [S1].

2.  **NeRF Optimization (Reconstruction):**
    For each iteration $i$ from 1 to 1000:
    1.  **Sample Input Views:** Select a subset of input images $x_{obs}$ and poses $\pi_{obs}$ [S1].
    2.  **Sample Novel Pose:** Select a target novel camera pose $\pi$ based on a distribution matching the capture pattern (e.g., nearest 3 poses from $\pi_{obs}$) [S1].
    3.  **Render NeRF:** Render an image $x$ from the current NeRF parameters $\psi$ at pose $\pi$ [S1].
    4.  **Encode to Latent:** Encode the rendered image $x$ into a latent representation using the VAE encoder $E$ [S1].
    5.  **Add Noise:** Add noise to the latent at timestep $t$ (where $t \in [t_{min}, t_{max}]$) to obtain a noisy latent $z_t$ [S1].
    6.  **Render Feature Map:** Compute the feature map $f = R_\phi(x_{obs}, \pi_{obs}, \pi)$ using the PixelNeRF encoder [S1].
    7.  **Diffusion Sampling:**
        a.  Condition the denoising U-Net $\epsilon_\theta$ with the noisy latent $z_t$, the feature map $f$, and CLIP embeddings of $x_{obs}$ [S1].
        b.  Perform $k=10$ steps of DDIM sampling to generate a denoised sample image $\hat{x}$ [S1].
        c.  Apply classifier-free guidance with scale 3.0 [S1].
    8.  **Compute Losses:**
        a.  **Reconstruction Loss ($L_{recon}$):** Compute the Charbonnier loss between the NeRF rendering $x$ and the observed image $x_{obs}$ at the corresponding observed pose [S1].
            $$ L_{recon}(\psi) = \mathbb{E}_{x_{obs}, \pi_{obs}} \ell(x(\psi, \pi_{obs}), x_{obs}) $$
            where $\ell$ is the Charbonnier loss [S1].
        b.  **Sampling Loss ($L_{sample}$):** Compute the loss between the generated sample $\hat{x}$ and the NeRF rendering $x$ at the novel pose $\pi$ [S1].
            $$ L_{sample}(\psi) = \mathbb{E}_{\pi, t} w(t) \left( \|x - \hat{x}_\pi\|_1 + L_p(x, \hat{x}_\pi) \right) $$
            where $L_p$ is the perceptual distance (LPIPS) [65], and $w(t)$ is a noise-level dependent weighting function [S1].
    9.  **Total Loss:**
        $$ L_{total} = L_{recon} + \lambda_{sample} L_{sample} $$
        where $\lambda_{sample}$ is linearly decayed from 1 to 0.1 over training [S1].
    10. **Update:** Backpropagate $L_{total}$ through the NeRF parameters $\psi$ and update them [S1].

### Edge Cases and Pitfalls
-   **Few-View Scenarios (3 and 9 views):** The reconstruction problem is severely underconstrained. The diffusion prior prevents catastrophic failures and improves robustness [S1].
-   **Dense Capture (e.g., 54 views):** The method helps reduce "fog" and "floater" artifacts common to volumetric reconstruction methods like NeRF while preserving the appearance of well-sampled regions [S1].
-   **Novel View Selection:** Views should not be placed inside objects or behind walls. The pose distribution should match the expected observation positions [S1].

## Operating characteristics
-   **Time Complexity:** The NeRF is trained for 1000 iterations [S1]. Each iteration involves rendering from the NeRF, encoding to latent space, adding noise, rendering a feature map, and performing 10 steps of DDIM sampling with the diffusion model [S1].
-   **Memory Complexity:** Depends on the resolution ($512 \times 512$) and the latent space dimensions ($64 \times 64 \times 8$) [S1].
-   **Performance:** Outperforms existing baselines on forward-facing and unbounded 360-degree scenes [S1]. Empirically found to work better than SDS [S1].
-   **Strengths:** Effective drop-in regularizer for NeRFs across a range of capture settings; synthesizes realistic geometry and texture in underconstrained regions; preserves appearance of observed regions [S1].
-   **Limitations:** Requires a pretrained diffusion model finetuned on multiview data; performance depends on the quality of the diffusion prior [S1].
-   **When to use:** When reconstructing real-world scenes from a small number of input images (few-view) [S1].
-   **When not to use:** When dense input views are available and artifacts are minimal, the overhead of the diffusion prior may be unnecessary [S1].

## Background
ReconFusion was introduced in the paper "ReconFusion: 3D Reconstruction with Diffusion Priors" [S1]. It addresses the limitation of NeRF methods that require tens to hundreds of input images for high-quality reconstruction [S1]. The method builds on the concept of using diffusion models as priors for 3D generation, similar to DreamFusion [36] and SparseFusion [67], but specifically conditions the diffusion model on multiple posed input images to enable few-view reconstruction [S1]. The diffusion model is finetuned from a pretrained LDM on a mixture of real-world and synthetic multiview datasets [S1].

## Sources
[S1] ReconFusion: 3D Reconstruction with Diffusion Priors, https://doi.org/10.1109/cvpr52733.2024.02036, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
