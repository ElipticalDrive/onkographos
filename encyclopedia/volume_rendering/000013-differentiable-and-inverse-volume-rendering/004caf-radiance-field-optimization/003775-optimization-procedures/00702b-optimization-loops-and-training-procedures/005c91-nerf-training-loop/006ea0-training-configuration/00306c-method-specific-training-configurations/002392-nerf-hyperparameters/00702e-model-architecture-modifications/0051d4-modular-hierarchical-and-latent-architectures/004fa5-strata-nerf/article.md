# Strata-NeRF

Also called: Neural Radiance Fields for Stratified Scenes

## Overview
Strata-NeRF is a neural radiance field method designed to implicitly capture scenes with multiple hierarchical levels (stratified scenes) by conditioning the radiance field on Vector Quantized (VQ) latent representations [S1]. Unlike standard continuous NeRFs, which produce artifacts like haziness and white clouds when trained jointly on images from different structural levels, Strata-NeRF uses discrete latent codes to model sudden changes and discontinuities in scene structure [S1]. It achieves this through a Latent Generator that compresses 3D positions into discrete codes and a Latent Router that conditions the radiance field MLP, allowing a single network to represent complex, multi-tiered environments such as buildings with distinct interior and exterior geometries [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   3D spatial position $\mathbf{x} \in \mathbb{R}^3$.
    *   Viewing direction $\mathbf{d} \in \mathbb{R}^3$.
    *   Camera pose parameters (implicitly handled via the ray generation process, consistent with the base Mip-NeRF 360 architecture).
    *   Ground truth color $c^*(\mathbf{r})$ for reconstruction loss.
*   **Outputs:**
    *   Volume density $\sigma$.
    *   View-dependent color $\mathbf{c} \in \mathbb{R}^3$.

### Data Structures

**1. Vector Codebook**
*   **Layout:** A lookup table of discrete latent vectors.
*   **Size:** $N = 1024$ entries.
*   **Dimension:** $D = 48$ per vector.
*   **Type:** Float32 (standard for VQ-VAE codebooks).
*   **Access:** Nearest neighbor search based on Euclidean distance.

**2. Latent Generator Encoder**
*   **Structure:** Two linear layers.
*   **Hidden Size:** 48.
*   **Input:** Integrated Positional Encoded (IPE) position $\boldsymbol{\gamma}(\mathbf{x})$.
*   **Output:** Continuous latent vector $\mathbf{z} \in \mathbb{R}^{48}$.

**3. Latent Generator Decoder**
*   **Structure:** One linear layer.
*   **Hidden Size:** 96.
*   **Input:** Quantized latent vector $\mathbf{z}_e \in \mathbb{R}^{48}$.
*   **Output:** Reconstructed position $\mathbf{y} \in \mathbb{R}^{48}$ (matching IPE dimension).

**4. Latent Router**
*   **Structure:** Two linear layers.
*   **Hidden Size:** 256.
*   **Input:** Quantized latent vector $\mathbf{z}_e$.
*   **Output:** Conditioning signal applied to the first two layers of the Radiance Field MLP.

**5. Radiance Field MLP (Base)**
*   **Architecture:** Based on Mip-NeRF 360 [S1].
*   **Conditioning:** The first two layers are conditioned by the output of the Latent Router.

### Numbered Steps

1.  **Positional Encoding:** For a given 3D point $\mathbf{x}$ along a ray, compute the Integrated Positional Encoding (IPE) $\boldsymbol{\gamma}(\mathbf{x})$ [S1]. This matches the input dimension of the Latent Generator.
2.  **Latent Generation:** Pass $\boldsymbol{\gamma}(\mathbf{x})$ through the Latent Generator Encoder to obtain a continuous latent vector $\mathbf{z}$:
    $$ \mathbf{z} = \text{Encoder}_{\text{LG}}(\boldsymbol{\gamma}(\mathbf{x})) $$
3.  **Vector Quantization:** Find the nearest neighbor in the Codebook $\mathbf{E} \in \mathbb{R}^{N \times D}$ to $\mathbf{z}$. Let $\mathbf{z}_e$ be the selected codebook vector:
    $$ \mathbf{z}_e = \arg \min_{\mathbf{e}_i \in \mathbf{E}} ||\mathbf{z} - \mathbf{e}_i||_2 $$
    *Note: During training, the gradient flows through $\mathbf{z}$ to the encoder and through $\mathbf{z}_e$ to the codebook using stop-gradient operators as defined in the loss.*
4.  **Latent Routing:** Pass the quantized code $\mathbf{z}_e$ through the Latent Router MLP to produce a conditioning vector.
5.  **Radiance Field Conditioning:** Inject the conditioning vector from the Latent Router into the first two layers of the Radiance Field MLP [S1].
6.  **Density and Color Prediction:** Feed the spatial position $\mathbf{x}$, viewing direction $\mathbf{d}$, and the conditioned features into the rest of the Radiance Field MLP to predict volume density $\sigma$ and color $\mathbf{c}$ [S1].
7.  **Volume Rendering:** Integrate $\sigma$ and $\mathbf{c}$ along the ray using the standard differentiable volume rendering equation (as defined in the parent article [Parent](3065-nerf-hyperparameters.md)) to produce the final pixel color $c(\mathbf{r}, t)$ [S1].
8.  **Loss Calculation:** Compute the total loss $L_{total}$ (see Formulas) and backpropagate to update all parameters (Latent Generator, Codebook, Latent Router, and Radiance Field) [S1].

### Formulas

**1. Latent Generator VQ Loss ($L_{vq}$)**
This loss trains the Latent Generator (VAE block) to reconstruct the encoded position and align with the codebook [S1]:
$$ L_{vq} = ||\boldsymbol{\gamma}(\mathbf{x}) - \mathbf{y}||_2^2 + ||sg(\mathbf{z}_e) - \mathbf{z}||_2^2 + \beta||\mathbf{z}_e - sg(\mathbf{z})||_2^2 $$
Where:
*   $\boldsymbol{\gamma}(\mathbf{x})$: Integrated Positional Encoded input.
*   $\mathbf{y}$: Reconstructed output from the Decoder.
*   $\mathbf{z}$: Continuous latent output from the Encoder.
*   $\mathbf{z}_e$: Quantized latent code from Codebook $\mathbf{E}$.
*   $sg(\cdot)$: Stop-gradient operator.
*   $\beta$: Commitment loss weight.

**2. Total Training Loss ($L_{total}$)**
$$ L_{total} = L_{recon}(c(\mathbf{r}, t), c^*(\mathbf{r})) + \lambda_1 L_{dist}(\mathbf{s}, \mathbf{w}) + \lambda_2 L_{vq} $$
Where:
*   $L_{recon}$: Photometric reconstruction loss between rendered color $c(\mathbf{r}, t)$ and ground truth $c^*(\mathbf{r})$.
*   $L_{dist}$: Distortion loss between normalized ray distances $\mathbf{s}$ and weight vector $\mathbf{w}$ [S1].
*   $L_{vq}$: The VQ loss defined above.
*   $\lambda_1, \lambda_2$: Loss weights.

### Parameters

*   **$\lambda_1$ (Distortion Loss Weight):** 0.01 [S1].
*   **$\lambda_2$ (VQ Loss Weight):** 0.1 [S1].
*   **$\beta$ (Commitment Loss Weight):** 1.0 [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $1 \times 10^{-6}$ [S1].
*   **Training Iterations:** 150,000 per scene [S1].
*   **Codebook Size:** 1024 vectors [S1].
*   **Codebook Dimension:** 48 [S1].
*   **Encoder Hidden Size:** 48 [S1].
*   **Decoder Hidden Size:** 96 [S1].
*   **Router Hidden Size:** 256 [S1].

### Edge Cases and Pitfalls

*   **Continuous Artifacts:** Standard NeRFs (e.g., Mip-NeRF 360) trained jointly on stratified scenes produce "cloudy" or "sub-optimal" views with white artifacts and haziness because their continuous nature cannot model sudden structural changes between levels [S1]. Strata-NeRF mitigates this by using discrete VQ latents [S1].
*   **Discontinuity Modeling:** The method explicitly relies on the discreteness of VQ codes to handle discontinuities. If the codebook is too small or the quantization is too aggressive, the model may fail to distinguish between distinct hierarchical levels [S1].
*   **Base Architecture Dependency:** Strata-NeRF builds on Mip-NeRF 360. It does not alter the proposal MLP or the core volume rendering integration logic, relying instead on the conditioning mechanism [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class StrataNeRF:
    def __init__(self, codebook_size=1024, codebook_dim=48, ipe_dim=48):
        """
        Initialize Strata-NeRF components.
        """
        # Codebook: N x D
        self.codebook = nn.Embedding(codebook_size, codebook_dim)
        # Initialize codebook weights
        nn.init.uniform_(self.codebook.weight, -1/codebook_dim, 1/codebook_dim)
        
        # Latent Generator Encoder: 2 linear layers, hidden 48
        self.encoder = nn.Sequential(
            nn.Linear(ipe_dim, 48),
            nn.ReLU(),
            nn.Linear(48, codebook_dim)
        )
        
        # Latent Generator Decoder: 1 linear layer, hidden 96
        self.decoder = nn.Sequential(
            nn.Linear(codebook_dim, 96),
            nn.ReLU(),
            nn.Linear(96, ipe_dim)
        )
        
        # Latent Router: 2 linear layers, hidden 256
        self.router = nn.Sequential(
            nn.Linear(codebook_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 256) # Output size matches conditioning dim for first 2 layers of RF
        )
        
        # Radiance Field MLP (Base Mip-NeRF 360)
        # Note: The specific architecture of the RF is assumed to be from Mip-NeRF 360.
        # The first two layers are conditioned by the router output.
        self.radiance_field = MipNeRF360Base() 

    def forward(self, x, d):
        """
        Forward pass for a single 3D point x and direction d.
        """
        # 1. Positional Encoding (Integrated PE)
        # Assuming get_ipe is a helper that computes IPE for x
        gamma_x = get_ipe(x) 
        
        # 2. Latent Generation
        z = self.encoder(gamma_x)
        
        # 3. Vector Quantization
        # Compute distances to all codebook entries
        # z shape: (B, D), codebook weight shape: (N, D)
        distances = torch.cdist(z, self.codebook.weight) # (B, N)
        indices = torch.argmin(distances, dim=1)
        
        # Get quantized codes
        z_e = self.codebook(indices)
        
        # 4. Latent Routing
        # Condition the radiance field
        router_output = self.router(z_e)
        
        # 5. Radiance Field Prediction
        # The RF takes x, d, and the router_output as conditioning
        sigma, c = self.radiance_field(x, d, conditioning=router_output)
        
        return sigma, c, z, z_e, gamma_x

    def compute_loss(self, rendered_color, gt_color, sigma, weights, normalized_distances, 
                     gamma_x, z, z_e, beta=1.0, lambda1=0.01, lambda2=0.1):
        """
        Compute total loss.
        """
        # Reconstruction Loss
        l_recon = F.mse_loss(rendered_color, gt_color)
        
        # Distortion Loss (from Mip-NeRF 360)
        l_dist = compute_distortion_loss(normalized_distances, weights)
        
        # VQ Loss
        # Decoder reconstruction
        y = self.decoder(z_e)
        recon_loss = torch.norm(gamma_x - y, p=2, dim=1).mean()
        
        # Commitment loss and codebook loss
        # sg denotes stop-gradient
        z_e_stopped = z_e.detach()
        z_stopped = z.detach()
        
        commitment_loss = torch.norm(z_e_stopped - z, p=2, dim=1).mean()
        codebook_loss = torch.norm(z_e - z_stopped, p=2, dim=1).mean()
        
        l_vq = recon_loss + commitment_loss + beta * codebook_loss
        
        # Total Loss
        l_total = l_recon + lambda1 * l_dist + lambda2 * l_vq
        
        return l_total, l_recon, l_dist, l_vq

# Helper for Mip-NeRF 360 base architecture details would go here
# This is a placeholder to satisfy the "self-contained" requirement for the RF part
class MipNeRF360Base(nn.Module):
    def __init__(self):
        super().__init__()
        # Placeholder for the actual Mip-NeRF 360 MLP structure
        # The key modification is that the first two layers receive conditioning
        pass
    def forward(self, x, d, conditioning=None):
        # In Strata-NeRF, conditioning is added to the first two layers
        # This is a simplified representation
        return 0.1, torch.zeros(3) # Dummy output
```

## Subtopics

### Mip-NeRF 360
The base radiance field architecture used by Strata-NeRF. It provides the continuous volumetric representation and volume rendering pipeline that Strata-NeRF conditions. [Link](3065-nerf-hyperparameters.md)

### VQ-VAE
The underlying mechanism for Vector Quantization used in the Latent Generator. It learns a discrete codebook to model the latent space of 3D positions. [Link](14567-modular-hierarchical-and-latent-architectures.md) (See "Representation Basis" section in parent)

## Operating characteristics

*   **Time Complexity:** Training time is comparable to Mip-NeRF 360 but requires 150k iterations. The addition of the Latent Generator and Router adds minimal computational overhead per step as they are lightweight MLPs [S1].
*   **Memory Complexity:** The method introduces significantly fewer parameters than training independent NeRF models for each level, leading to reduced memory usage [S1]. The codebook size (1024x48) is negligible compared to the MLP weights.
*   **Measured Performance:** Evaluated on a synthetic Stratified Scenes dataset (simple and complex geometry) and the real-world RealEstate10K dataset. Strata-NeRF produces high-fidelity views and reduces artifacts (haziness, white clouds) compared to Mip-NeRF 360 when trained jointly on stratified levels [S1].
*   **Strengths:**
    *   Handles sudden structural changes and discontinuities in stratified scenes.
    *   Single model for multi-level scenes.
    *   Reduces memory footprint compared to per-level models.
*   **Limitations:**
    *   Relies on the quality of the VQ codebook; poor quantization may blur level distinctions.
    *   Evaluated primarily on specific stratified datasets; generalization to non-stratified scenes is not the primary focus (though it functions as a conditional NeRF).
*   **When to Use:** Use when rendering scenes with distinct, non-continuous hierarchical levels (e.g., interior/exterior of a building) where standard continuous NeRFs fail.
*   **When Not to Use:** Use standard NeRF or Mip-NeRF 360 for scenes without significant structural discontinuities or hierarchical stratification.

## Background

Strata-NeRF was proposed in 2023 by Ati et al. [S1]. It addresses the limitation of existing NeRF methods in modeling "stratified scenes," which are defined as scenes deconstructible into several tiers or groupings of structure (e.g., exterior and interior levels of a building) [S1]. Previous methods like Mip-NeRF 360 produced unsatisfactory results (cloudy views, artifacts) when trained jointly on such scenes due to their continuous nature [S1]. Strata-NeRF introduces the concept of using Vector Quantized (VQ) latent representations to condition the NeRF, allowing for discrete switching between scene structures [S1]. The method was evaluated on a newly proposed synthetic Stratified Scenes dataset and the RealEstate10K dataset [S1].

## Sources

[S1] Strata-NeRF : Neural Radiance Fields for Stratified Scenes, https://arxiv.org/abs/2308.10337v1, fetched 2026-10-26
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
