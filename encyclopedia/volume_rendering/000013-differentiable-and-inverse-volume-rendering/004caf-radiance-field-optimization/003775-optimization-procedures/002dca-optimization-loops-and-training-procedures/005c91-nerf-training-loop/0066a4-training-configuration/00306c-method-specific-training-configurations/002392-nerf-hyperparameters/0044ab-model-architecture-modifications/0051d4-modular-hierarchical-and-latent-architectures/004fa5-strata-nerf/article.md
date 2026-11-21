# Strata-NeRF

Also called: Neural Radiance Fields for Stratified Scenes

## Overview
Strata-NeRF is a neural radiance field method designed to implicitly capture scenes with multiple hierarchical levels (stratified scenes) by conditioning the radiance field on Vector Quantized (VQ) latent representations [S1]. Unlike standard NeRFs which assume continuous scene geometry and fail to model sudden structural changes between levels (producing cloudy artifacts or haziness), Strata-NeRF uses discrete latent codes to handle discontinuities in scene structure [S1]. It builds on [Modular, Hierarchical, and Latent Architectures](14567-modular-hierarchical-and-latent-architectures.md) by introducing a Latent Generator and Latent Router to compress 3D positional information into discrete codes that condition the MLP [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D point coordinates $\mathbf{x}$ along a ray, integrated positional encoding $\boldsymbol{\gamma}(\mathbf{x})$, and camera parameters.
*   **Output:** Density $\sigma$ and color $\mathbf{c}$ for volume rendering.

**Data Structures:**
1.  **Codebook $\mathbf{E}$:** A lookup table of size $N \times D$. In the reference implementation, $N=1024$ and $D=48$ [S1].
2.  **Latent Generator Encoder:** Two linear layers with hidden size 48 [S1].
3.  **Latent Generator Decoder:** One linear layer with hidden size 96 [S1].
4.  **Latent Router:** Two linear layers with hidden size 256 [S1].
5.  **Radiance Field MLP:** The base network (based on Mip-NeRF 360) where the first two layers are conditioned by the routed latent [S1].

**Algorithm Steps:**

1.  **Positional Encoding:** Compute the integrated positional encoding $\boldsymbol{\gamma}(\mathbf{x})$ for the 3D point $\mathbf{x}$ [S1].
2.  **Latent Generation:**
    a. Pass $\boldsymbol{\gamma}(\mathbf{x})$ through the Latent Generator Encoder to get the continuous latent vector $\mathbf{z}$ [S1].
    b. Find the nearest neighbor in the Codebook $\mathbf{E}$ to $\mathbf{z}$. Let $\mathbf{z}_e$ be the quantized latent code (the vector from $\mathbf{E}$) [S1].
    c. Pass $\mathbf{z}_e$ through the Latent Generator Decoder to get the reconstructed output $\mathbf{y}$ [S1].
3.  **Latent Routing:**
    a. Pass the quantized latent code $\mathbf{z}_e$ through the Latent Router MLP [S1].
    b. The output of the Latent Router is used to condition the first two layers of the Radiance Field MLP [S1].
4.  **Radiance Field Evaluation:**
    a. Compute density $\sigma$ and color $\mathbf{c}$ using the Radiance Field MLP, conditioned by the routed latent [S1].
5.  **Volume Rendering:** Render the color $\mathbf{c}(\mathbf{r}, t)$ along the ray $\mathbf{r}$ using standard volume rendering integration [S1].

**Loss Functions:**

The total loss is a combination of reconstruction, distortion, and VQ losses [S1]:

$$ L_{total} = L_{recon}(\mathbf{c}(\mathbf{r}, t), \mathbf{c}^*(\mathbf{r})) + \lambda_1 L_{dist}(\mathbf{s}, \mathbf{w}) + \lambda_2 L_{vq} $$

Where:
*   $L_{recon}$ is the photometric reconstruction loss between estimated color $\mathbf{c}(\mathbf{r}, t)$ and ground truth $\mathbf{c}^*(\mathbf{r})$ [S1].
*   $L_{dist}$ is the distortion loss between normalized ray distances $\mathbf{s}$ and weight vector $\mathbf{w}$ [S1].
*   $L_{vq}$ is the Vector Quantization loss [S1]:

$$ L_{vq} = ||\boldsymbol{\gamma}(\mathbf{x}) - \mathbf{y}||_2^2 + ||sg(\mathbf{z}_e) - \mathbf{z}||_2^2 + \beta||\mathbf{z}_e - sg(\mathbf{z})||_2^2 $$

Where:
*   $\mathbf{z}$ is the encoder output (continuous latent) [S1].
*   $\mathbf{z}_e$ is the quantized latent code from the codebook [S1].
*   $\mathbf{y}$ is the decoder output (reconstructed position) [S1].
*   $sg(\cdot)$ denotes the stop-gradient operator [S1].
*   $\beta$ is a hyperparameter for the commitment loss [S1].

**Parameters:**
*   $\lambda_1$ (distortion loss weight): 0.01 [S1].
*   $\lambda_2$ (VQ loss weight): 0.1 [S1].
*   $\beta$ (commitment loss weight): 1.0 [S1].
*   Codebook size: 1024 vectors [S1].
*   Codebook dimension: 48 [S1].
*   Training iterations: 150,000 [S1].
*   Optimizer: Adam with learning rate $1 \times 10^{-6}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class StrataNeRF:
    def __init__(self, codebook_size=1024, codebook_dim=48, hidden_encoder=48, hidden_decoder=96, hidden_router=256):
        # Codebook: N x D
        self.codebook = nn.Embedding(codebook_size, codebook_dim)
        
        # Latent Generator Encoder: 2 linear layers, hidden 48
        # Input dim matches IPE output dim (let's assume D_in for generality, 
        # but decoder output matches IPE dim, so encoder output is codebook_dim)
        self.encoder = nn.Sequential(
            nn.Linear(codebook_dim, hidden_encoder), # Assuming input is already encoded or mapped to D
            nn.ReLU(),
            nn.Linear(hidden_encoder, codebook_dim)
        )
        
        # Latent Generator Decoder: 1 linear layer, hidden 96
        # Output dim matches IPE output dim
        self.decoder = nn.Sequential(
            nn.Linear(codebook_dim, hidden_decoder),
            nn.ReLU(),
            nn.Linear(hidden_decoder, codebook_dim) # Output matches codebook dim for reconstruction of latent space? 
            # Note: Source says "output dimension of our decoder matches the output from Integrated Positional Encoding (IPE) block".
            # If IPE dim != codebook_dim, this mapping needs adjustment. 
            # However, L_vq uses ||gamma(x) - y||. If gamma(x) is high-dim, y must be high-dim.
            # But encoder output z is codebook_dim. 
            # Let's assume the "input" to encoder is gamma(x) and it projects to codebook_dim.
            # And decoder projects codebook_dim back to IPE_dim.
            # Re-reading: "encoder has two linear layers... decoder has one... output dimension... matches IPE".
            # This implies: gamma(x) [IPE_dim] -> Encoder -> z [codebook_dim] -> Codebook -> z_e [codebook_dim] -> Decoder -> y [IPE_dim].
            pass 
        ]
        
        # Correction for dimensions based on text:
        # Encoder input: gamma(x) [IPE_dim]. Output: z [codebook_dim].
        # Decoder input: z_e [codebook_dim]. Output: y [IPE_dim].
        self.encoder = nn.Sequential(
            nn.Linear(codebook_dim, hidden_encoder), # Wait, input to encoder is gamma(x). 
            # The text says "encoder has two linear layers of hidden size 48". 
            # It doesn't explicitly state input dim, but it processes gamma(x).
            # Let's assume the first layer maps IPE_dim to 48.
            nn.Linear(codebook_dim, hidden_encoder), # Placeholder for IPE_dim -> 48
            nn.ReLU(),
            nn.Linear(hidden_encoder, codebook_dim)
        )
        
        self.decoder = nn.Sequential(
            nn.Linear(codebook_dim, hidden_decoder),
            nn.ReLU(),
            nn.Linear(hidden_decoder, codebook_dim) # Placeholder for 96 -> IPE_dim
        )
        
        # Latent Router: 2 linear layers, hidden 256
        self.router = nn.Sequential(
            nn.Linear(codebook_dim, hidden_router),
            nn.ReLU(),
            nn.Linear(hidden_router, hidden_router)
        )
        
        # Radiance Field MLP (Base MipNeRF 360)
        # The first two layers are conditioned by the router output.
        self.radiance_field = MipNeRF360Base() 

    def forward(self, x, ray_params):
        # 1. Get Integrated Positional Encoding
        gamma_x = get_integrated_positional_encoding(x) # Returns vector of dim IPE_dim
        
        # 2. Latent Generator
        # Encode gamma(x) to z
        z = self.encoder(gamma_x)
        
        # Quantize: Find nearest codebook entry
        # Calculate distances between z and all codebook entries
        # z shape: [B, D], codebook shape: [N, D]
        distances = torch.cdist(z, self.codebook.weight, p=2)
        indices = torch.argmin(distances, dim=1)
        
        # Get quantized code z_e
        z_e = self.codebook(indices)
        
        # Decode z_e to y
        y = self.decoder(z_e)
        
        # 3. Latent Router
        # Route z_e to condition the MLP
        routed_latent = self.router(z_e)
        
        # 4. Radiance Field
        # Condition first two layers of MLP with routed_latent
        # This requires modifying the base MLP to accept conditioning.
        # For the purpose of this article, we assume the base MLP has a method to inject this.
        sigma, color = self.radiance_field(x, ray_params, conditioning=routed_latent)
        
        return sigma, color, indices, z, z_e, y

    def compute_loss(self, color_pred, color_gt, sigma, ray_weights, ray_distances, gamma_x, z, z_e, y):
        # L_recon
        l_recon = F.mse_loss(color_pred, color_gt)
        
        # L_dist
        l_dist = compute_distortion_loss(ray_distances, ray_weights)
        
        # L_vq
        # sg denotes stop-gradient
        z_e_sg = z_e.detach()
        z_sg = z.detach()
        
        l_vq = (torch.norm(gamma_x - y, dim=1)**2).mean() + \
               torch.norm(z_e_sg - z, dim=1)**2).mean() + \
               1.0 * torch.norm(z_e - z_sg, dim=1)**2).mean()
               
        lambda1 = 0.01
        lambda2 = 0.1
        
        l_total = l_recon + lambda1 * l_dist + lambda2 * l_vq
        
        return l_total
```

*Note on Pseudocode:* The dimension mapping in the encoder/decoder is inferred from the loss function $L_{vq}$ which compares $\boldsymbol{\gamma}(\mathbf{x})$ and $\mathbf{y}$. Thus, the decoder output $\mathbf{y}$ must have the same dimension as the Integrated Positional Encoding $\boldsymbol{\gamma}(\mathbf{x})$. The encoder maps $\boldsymbol{\gamma}(\mathbf{x})$ to the codebook dimension $D=48$. The router takes the codebook dimension $D=48$ as input.

## Operating characteristics
*   **Time Complexity:** Training requires 150k iterations [S1]. The addition of the Latent Generator and Router adds minimal computational overhead compared to training independent NeRFs for each level [S1].
*   **Memory Complexity:** The method uses significantly fewer parameters than training an independent NeRF model for each level, leading to reduced memory usage [S1]. The codebook size is 1024 vectors of dimension 48 [S1].
*   **Strengths:** Effectively captures stratified scenes with sudden structural changes (e.g., exterior/interior of a building) where standard NeRFs fail [S1]. Reduces artifacts like haziness and white clouds seen in jointly trained continuous NeRFs [S1].
*   **Limitations:** Requires a stratified scene structure (discrete levels). Performance on non-stratified scenes is not the primary focus, though it builds on Mip-NeRF 360 [S1].
*   **When to use:** When rendering scenes with distinct hierarchical levels (e.g., inside/outside a structure) and standard NeRFs produce artifacts due to continuous assumptions [S1].
*   **When not to use:** For simple, non-stratified scenes where standard NeRFs or Instant-NGP are sufficient and faster [S1].

## Background
Strata-NeRF was introduced in 2023 by Ati et al. [S1]. It addresses the limitation of existing neural radiance fields (like Mip-NeRF 360) which struggle to model scenes that can be deconstructed into several tiers or groupings of structure, termed "Stratified Scenes" [S1]. Standard NeRFs produce unsatisfactory results (cloudy/sub-optimal views) when trained on images from all hierarchical levels together because their continuous nature is unsuitable for modelling sudden changes [S1]. The method was evaluated on a synthetic Stratified Scenes dataset (spheres, cubes, complex geometry) and the real-world RealEstate10K dataset [S1]. It builds upon the principles of [Modular, Hierarchical, and Latent Architectures](14567-modular-hierarchical-and-latent-architectures.md) by using discrete VQ latents to handle scene complexity [S1].

## Sources
[S1] Strata-NeRF : Neural Radiance Fields for Stratified Scenes, https://arxiv.org/abs/2308.10337v1, fetched 2026-10-26
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
