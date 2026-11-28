# MaterialFusion

Also called: Prior-guided Inverse Rendering

## Overview
MaterialFusion is a 3D inverse rendering pipeline that reconstructs an object’s geometry, physically-based rendering (PBR) material properties, and environmental illumination from multi-view images under unknown lighting conditions [S1]. It addresses the intrinsic challenge of disentangling albedo and material properties from input images, which typically leads to poor relighting performance in novel conditions [S1]. Unlike standard inverse rendering methods that optimize geometry and materials solely based on photometric consistency, MaterialFusion incorporates a 2D diffusion prior (StableMaterial) to guide the optimization of albedo and material textures, ensuring physically plausible and disentangled results [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ multi-view RGB images $\{x_i\}$ of an object captured under unknown, varying illumination [S1].
*   **Outputs:** A 3D mesh geometry $G$, UV material textures (albedo $k_d$ and ORM $k_{orm}$), and an HDR environment light map $L$ [S1].

**Data Structures and Layouts:**
1.  **Geometry:** Parameterized as a Signed Distance Field (SDF) $S$ [S1]. The mesh $G$ is extracted from $S$ in each optimization iteration using DMTet [S1].
2.  **Material Texture:**
    *   **Albedo ($k_d$):** A UV-map where each texel is $a \in \mathbb{R}^3$ [S1].
    *   **ORM ($k_{orm}$):** A UV-map where each texel is $(o, r, m)$ [S1]. $o$ (occlusion) is unused [S1]. $r \in \mathbb{R}$ is roughness, and $m \in \mathbb{R}$ is metallicness [S1].
3.  **Illumination:** Represented as a High Dynamic Range (HDR) environment map $L$ [S1].

**Optimization Steps:**
The method optimizes $S$, $k_d$, $k_{orm}$, and $L$ by minimizing a total loss function [S1]. The procedure for each optimization iteration is as follows:

1.  **Sampling:** Sample a subset of training images $x$ and their associated camera views $C$ [S1].
2.  **Rendering:**
    *   Extract mesh $G$ from SDF $S$ using DMTet [S1].
    *   Use the differentiable renderer (nvdiffrecmc) with inputs $G$, $k_d$, $k_{orm}$, $C$, and $L$ to produce:
        *   The rendered RGB image $\hat{x}$ [S1].
        *   The rendered albedo component $\hat{I}_d$ [S1].
        *   The rendered ORM component $\hat{I}_{orm}$ [S1].
3.  **Diffusion Prior Encoding:**
    *   Encode $\hat{I}_d$ and $\hat{I}_{orm}$ using the Stable Diffusion encoder to obtain their latent representations [S1].
    *   Concatenate these latents to form the combined latent $z$ [S1].
4.  **Diffusion Prior Denoising (SDS Calculation):**
    *   Sample Gaussian noise $\epsilon$ [S1].
    *   Add noise to $z$ according to a random diffusion timestep $t \in [0.02, 0.98]$ to obtain $z_t$ [S1].
    *   Denoise $z_t$ using the Stable Diffusion model via DDIM sampling for 5 steps, conditioned on the reference image $x$, to sample the clean latent $\hat{z}$ [S1].
5.  **Loss Computation:**
    *   Compute Reconstruction Loss $L_{recon}$ [S1].
    *   Compute SDS Loss $L_{SDS+}$ [S1].
    *   Compute Regularization Loss $L_{reg}$ [S1].
6.  **Update:** Minimize the total loss $L_{MaterialFusion}$ with respect to $S$, $k_d$, $k_{orm}$, and $L$ [S1].

**Formulas:**

*   **Reconstruction Loss:**
    $$L_{recon} = \mathbb{E}_C [L_2(\hat{x}, x)]$$
    where $\hat{x}$ is the rendered image and $x$ is the reference RGB image [S1].

*   **SDS Loss:**
    $$L_{SDS+} = \mathbb{E}_{t,\epsilon,v} [\lambda_{latent} \|z - \hat{z}\|_2^2 + \lambda_{rgb} \|D(z) - D(\hat{z})\|_2^2]$$
    where $D$ is the Stable Diffusion decoder, $z$ is the encoded latent of the rendered materials, and $\hat{z}$ is the denoised latent [S1].

*   **Total Loss:**
    $$L_{MaterialFusion} = L_{recon} + L_{reg} + \gamma_i L_{SDS+}$$
    where $\gamma_i$ is an iteration-dependent hyperparameter that decays as training progresses [S1].

**Parameters:**
*   **Diffusion Model:** Stable Diffusion 2.1 [S1].
*   **DDIM Sampling Steps:** 5 steps [S1].
*   **Timestep Range:** $t \in [0.02, 0.98]$ [S1].
*   **Regularization:** Uses the same $L_{reg}$ as nvdiffrecmc [S1].
*   **Glass Handling:** Glass surfaces in training objects are replaced with a black surface of roughness 0.25 and metallicness 0 [S1].
*   **Training Data Generation:** 30 images per object, with azimuth $\sim [0^\circ, 360^\circ]$ and elevation $\sim [-15^\circ, 90^\circ]$ on a hemisphere with radius $\sim [1.5, 2.0]$ [S1]. Lighting varies using StreetLearn, Laval, point lights, or directional sun lights [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def material_fusion_step(S, k_d, k_orm, L, x, C, diffusion_model, sd_encoder, sd_decoder, dmtet_extractor, renderer, lambda_latent, lambda_rgb, current_iteration, total_iterations):
    # 1. Extract mesh from SDF
    G = dmtet_extractor(S)

    # 2. Render RGB, Albedo, and ORM
    x_hat = renderer.render_rgb(G, k_d, k_orm, C, L)
    I_d_hat = renderer.render_albedo(G, k_d, k_orm, C, L)
    I_orm_hat = renderer.render_orm(G, k_d, k_orm, C, L)

    # 3. Encode rendered materials
    z_d = sd_encoder(I_d_hat)
    z_orm = sd_encoder(I_orm_hat)
    z = np.concatenate([z_d, z_orm], axis=-1)

    # 4. Add noise
    t = np.random.uniform(0.02, 0.98)
    epsilon = np.random.randn(*z.shape)
    z_t = z + t * epsilon

    # 5. Denoise using DDIM (5 steps) conditioned on x
    z_hat = z_t
    for _ in range(5):
        noise_pred = diffusion_model.predict_noise(z_hat, t, condition=x)
        z_hat = z_hat - t * noise_pred  # Simplified DDIM update [G]

    # 6. Compute SDS Loss
    latent_loss = lambda_latent * np.linalg.norm(z - z_hat, ord=2)**2
    z_rgb = sd_decoder(z)
    z_hat_rgb = sd_decoder(z_hat)
    rgb_loss = lambda_rgb * np.linalg.norm(z_rgb - z_hat_rgb, ord=2)**2
    l_sds = latent_loss + rgb_loss

    # 7. Compute Reconstruction Loss
    l_recon = np.linalg.norm(x_hat - x, ord=2)**2

    # 8. Compute Regularization Loss
    l_reg = compute_nvdiffrecmc_regularization(S, k_d, k_orm, L)

    # 9. Compute Total Loss
    gamma_i = max(0.0, 1.0 - current_iteration / total_iterations)
    total_loss = l_recon + l_reg + gamma_i * l_sds

    return total_loss

def compute_nvdiffrecmc_regularization(S, k_d, k_orm, L):
    # Combines normal smoothness, albedo smoothness, ORM smoothness, normal perturbation, and demodulated lighting regularization [S1]
    # Implementation details are specific to nvdiffrecmc [19]
    return 0.0  # Placeholder for regularization computation
```

**Edge Cases and Pitfalls:**
*   **Glass Surfaces:** Glass surfaces can cause issues in training data generation; they should be replaced with black surfaces (roughness 0.25, metallicness 0) to ensure consistent material learning [S1].
*   **SDS Weight Decay:** The weight $\gamma_i$ on the SDS loss must decay as training progresses to preserve finer details that might be lost due to the encoder/decoder operations [S1].
*   **Denoising Steps:** Using fewer than 5 DDIM steps may result in less crisp and accurate material predictions [S1].

## Subtopics

### MaterialFusion Regularization & Rendering Setup
Rendering and optimization configuration for MaterialFusion, utilizing nvdiffrast for differentiable rasterization and applying multiple regularizers to stabilize material reconstruction [S1]. It renders appearance, albedo, ORM, diffuse lighting, and specular lighting using nvdiffrast [S1]. It feeds rasterized estimated albedo and ORM batches to StableMaterial, converting inputs to RGB format to ensure sRGB tonemapped image inputs are properly handled [S1]. It applies regularizers including normal smoothness, albedo smoothness, ORM smoothness, normal perturbation, and demodulated lighting regularization [S1]. It differs from standard inverse rendering by integrating a pretrained diffusion prior (StableMaterial) and relying on nvdiffrast's rasterization rather than ray tracing for appearance rendering [S1]. It serves as the core optimization setup for the MaterialFusion inverse rendering pipeline [S1]. sRGB tonemapping must be explicitly handled before feeding inputs to StableMaterial to prevent color space mismatches [S1].

## Operating characteristics
*   **Time Complexity:** The method is computationally expensive due to the iterative extraction of the mesh via DMTet, differentiable rendering, and the 5-step DDIM denoising process per optimization step [S1].
*   **Memory Complexity:** Requires storing the SDF, UV maps, HDR environment map, and the latent representations for the diffusion model [S1].
*   **Performance:** Improves relighting performance in novel conditions compared to conventional inverse rendering methods by ensuring disentangled and physically plausible material properties [S1].
*   **Strengths:** Produces crisp and accurate material predictions; effectively disentangles albedo and material properties; handles unknown illumination by recovering the HDR environment map [S1].
*   **Limitations:** Relies on the quality of the 2D diffusion prior; glass surfaces require special handling; training data generation is complex (requires synthetic data with known materials and lighting) [S1].
*   **When to use:** When high-quality relighting under novel lighting is required and the input consists of multi-view images under unknown illumination [S1].
*   **When not to use:** When real-time performance is critical, or when the input images have significant glass/transparent surfaces that are not pre-processed [S1].

## Background
MaterialFusion was introduced in the paper "MaterialFusion: Enhancing Inverse Rendering with Material Diffusion Priors" [S1]. It builds upon the nvdiffrecmc inverse rendering pipeline and leverages advancements in distilling 3D information from 2D diffusion models [S1]. The method was developed to address the limitations of previous inverse rendering approaches that failed to accurately render objects under new lighting conditions due to poor disentanglement of albedo and material properties [S1]. The 2D prior, StableMaterial, was trained on a curated dataset of ~12K artist-designed synthetic Blender objects called BlenderVault [S1].

## Sources
[S1] MaterialFusion: Enhancing Inverse Rendering with Material Diffusion Priors, https://doi.org/10.1109/3dv66043.2025.00079, fetched 2026-10-29
