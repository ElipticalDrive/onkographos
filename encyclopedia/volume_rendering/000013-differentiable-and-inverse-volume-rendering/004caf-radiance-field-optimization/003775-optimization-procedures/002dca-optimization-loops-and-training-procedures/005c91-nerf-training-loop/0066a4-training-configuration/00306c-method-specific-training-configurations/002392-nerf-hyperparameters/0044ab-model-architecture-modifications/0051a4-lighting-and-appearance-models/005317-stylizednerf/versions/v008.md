# StylizedNeRF

Also called: StyleNeRF

## Overview
StylizedNeRF (also known as StyleNeRF) is a method for high-resolution 3D-aware image synthesis that integrates a style-based generator into the Neural Radiance Field (NeRF) architecture [S2, unofficial]. It solves the problem of generating high-fidelity, view-consistent 3D objects with controllable appearance styles, addressing the limitations of standard NeRF which lacks explicit style conditioning and struggles with high-resolution details [S2, unofficial]. It sets itself apart from siblings like [Neural Pre-Integrated Lighting](13783-neural-pre-integrated-lighting-neural-pil.md) by using a GAN-based mutual learning strategy with a 2D stylization network, rather than optimizing for physical lighting parameters [S1]. Unlike [Spherical Harmonics NeRF](14480-spherical-harmonics-nerf.md), it does not decompose lighting into spherical harmonics but instead uses latent codes to modulate the radiance field's appearance [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 2D images (typically face datasets like FFHQ, AFHQ, CompCars) resized to a target resolution (e.g., $256^2$) [S2, unofficial].
*   **Outputs:** A 3D radiance field model capable of rendering novel views of the synthesized object with a specific style [S2, unofficial].

**Data Structures:**
1.  **Latent Code ($\mathbf{w}$):** A vector sampled from a standard Gaussian distribution $\mathcal{N}(0, I)$ [S2, unofficial].
2.  **Mapping Network:** A multi-layer perceptron (MLP) that maps $\mathbf{w}$ to a style vector $\mathbf{w}'$ [S2, unofficial].
3.  **Synthesis Network (NeRF++ Backbone):**
    *   **Foreground MLP:** Predicts density $\sigma$ and feature vectors for points within a unit sphere [S2, unofficial].
    *   **Background MLP:** Predicts density $\sigma$ for points outside the unit sphere using inverted sphere parameterization; has fewer parameters than the foreground MLP [S2, unofficial].
    *   **Shared Color MLP:** Predicts color $\mathbf{c}$ based on position, view direction, and style-modulated features [S2, unofficial].
4.  **Style-Conditioned Block:** Consists of an affine transformation layer and a $1 \times 1$ Convolution layer [S2, unofficial].

**Algorithm Steps:**
1.  **Latent Sampling:** Sample a latent code $\mathbf{w} \sim \mathcal{N}(0, I)$ [S2, unofficial].
2.  **Style Mapping:** Pass $\mathbf{w}$ through the Mapping Network to obtain the style vector $\mathbf{w}'$ [S2, unofficial].
3.  **Ray Sampling:** For a given camera pose, sample points $\mathbf{x}_i$ along the ray [S2, unofficial].
4.  **Feature Extraction:** Pass $\mathbf{x}_i$ through the Foreground or Background MLP to get density $\sigma_i$ and intermediate features [S2, unofficial].
5.  **Style Modulation:**
    *   Apply affine transformation to the style vector $\mathbf{w}'$ to generate modulation parameters [S2, unofficial].
    *   Modulate the convolution weights in the Style-Conditioned Block using these parameters [S2, unofficial].
    *   Demodulate the convolution output for stability [S2, unofficial].
    *   Apply Leaky ReLU activation [S2, unofficial].
6.  **Color Prediction:** Pass the modulated features and view direction $\mathbf{d}$ into the Shared Color MLP to predict color $\mathbf{c}_i$ [S2, unofficial].
7.  **Volume Rendering:** Compute the rendered color $C(r)$ using the standard volume rendering integral:
    $$ C(r) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, and $\delta_i$ is the distance between samples [S2, unofficial].
8.  **Mutual Learning Optimization:**
    *   Optimize the Stylized NeRF and a 2D Stylization Network simultaneously [S1].
    *   Use learnable latent codes to handle ambiguities in 2D stylized results, ensuring 3D geometry consistency [S1].
    *   Apply a NeRF path regularization to enforce 3D consistency [S2, unofficial].
    *   Use a non-saturating GAN objective with R1 regularization [S2, unofficial].

**Parameters:**
*   **Input Resolution:** Fixed at $32^2$ for the training process [S2, unofficial].
*   **Target Resolution:** Images are resized to $256^2$ for evaluation [S2, unofficial].
*   **Batch Size:** 64 images per batch [S2, unofficial].
*   **Training Steps:** $T_1 = 500k$, $T_2 = 5000k$, $T_3 = 25000k$ images [S2, unofficial].
*   **Activation:** Leaky ReLU [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Small Datasets:** If the dataset is too small (e.g., etFaces), stable training is difficult; fine-tuning from a pretrained model (e.g., FFHQ) is required [S2, unofficial].
*   **3D Inconsistency:** Without proper regularization (like the NeRF path regularization), the model may produce artifacts where shape and appearance change constantly as the camera moves [S2, unofficial].

## Subtopics

### [Physical Imaging Model with Spherical Harmonics](16376-physical-imaging-model-with-spherical-harmonics.md)
This method integrates a physically constrained imaging model into the NeRF optimization loop to decompose illumination into direct sunlight and ambient skylight, using spherical harmonics to model the latter. It differs from standard NeRF by explicitly separating albedo, surface normals, and lighting to provide auxiliary supervision, thereby reducing lighting ambiguity and improving reconstruction fidelity in shadowed or radiometrically inconsistent regions. [Physical Imaging Model with Spherical Harmonics](16376-physical-imaging-model-with-spherical-harmonics.md)

### [StyleRF-VolVis](16573-stylerf-volvis.md)
StyleRF-VolVis is a framework for editing the style (appearance) of a DVR scene represented by a NeRF model without losing original content (geometry) information. It supports both photorealistic (PSE) and non-photorealistic (NPSE) editing in a single pipeline by separating content and appearance, allowing explicit assignment of different styles to different DVR regions via a Palette Color Network (PCN) and an Unrestricted Color Network (UCN), while preserving view-dependent lighting. It differs from prior works by supporting both photorealistic and non-photorealistic editing in a single framework, whereas others focus on one or the other. [StyleRF-VolVis](16573-stylerf-volvis.md)

### [Non-Photorealistic Style Editing](16597-non-photorealistic-style-editing.md)
Non-Photorealistic Style Editing (NPSE) is a style transfer process for volume rendering that assigns specific styles to distinct regions of a scene using an unrestricted color network and Nearest Neighbor Feature Matching (NNFM) loss. Unlike StylizedNeRF which assigns one style to the whole scene, NPSE allows explicit assignment of different styles to different DVR regions. Unlike ARF and StylizedNeRF which discard view directions to ensure consistency, NPSE removes view direction input only in the Unrestricted Color Network (UCN) optimization to preserve lighting while ensuring consistency. [Non-Photorealistic Style Editing](16597-non-photorealistic-style-editing.md)

### [StyleNeRF Architecture Configuration](16789-stylenerf-architecture-configuration.md)
StyleNeRF extends the original Neural Radiance Field (NeRF) architecture by integrating a style-based generator to handle illumination changes and synthesize high-resolution images]. It differs from standard NeRF by introducing a low-dimensional appearance code and style-based MLPs, setting it apart from siblings like StylizedNeRF which may use different stylization mechanisms [14882-stylizednerf.md]. [StyleNeRF Architecture Configuration](16789-stylenerf-architecture-configuration.md)

## Operating characteristics
*   **Time Complexity:** Training involves mutual learning between the 3D NeRF and a 2D stylization network, which increases computational cost compared to standard NeRF [S1]. The use of NeRF++ (foreground/background split) adds complexity to the density prediction step [S2, unofficial].
*   **Memory Complexity:** The architecture uses two MLPs for density (foreground and background) and a shared MLP for color, along with a Mapping Network and Style-Conditioned Blocks [S2, unofficial]. The input resolution is fixed at $32^2$ during training, which limits memory usage during the initial stages, but the target resolution is $256^2$ [S2, unofficial].
*   **Measured Performance:** On datasets like FFHQ, AFHQ, and CompCars, StyleNeRF achieves high-quality and 3D-consistent results, outperforming baselines like HoloGAN, GARF, and $\pi$-GAN which suffer from severe artifacts or 3D inconsistency [S2, unofficial].
*   **Strengths:** High-resolution synthesis, 3D consistency, style controllability, and ability to handle unbounded backgrounds via NeRF++ [S2, unofficial].
*   **Limitations:** Requires large datasets for stable training; small datasets require fine-tuning from pretrained models [S2, unofficial]. Computationally intensive due to GAN training and mutual learning [S1][S2, unofficial].
*   **When to use:** When high-fidelity, 3D-consistent, stylized 3D object synthesis is required, particularly for face datasets.
*   **When not to use:** When computational resources are limited, or when the dataset is small and fine-tuning is not feasible.

## Background
StylizedNeRF (StyleNeRF) was introduced in the paper "StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis" [S2, unofficial]. It builds upon the original NeRF architecture and StyleGAN2, integrating style-based generators into 3D-aware models to enable high-resolution image synthesis with style control [S2, unofficial]. The method addresses the limitations of previous 3D-aware generators that struggled with 3D consistency and high-resolution details [S2, unofficial]. It is part of the broader family of NeRF-based 3D editing and synthesis methods [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis, https://doi.org/10.48550/arxiv.2110.08985, fetched 2026-10-26 (unofficial)
