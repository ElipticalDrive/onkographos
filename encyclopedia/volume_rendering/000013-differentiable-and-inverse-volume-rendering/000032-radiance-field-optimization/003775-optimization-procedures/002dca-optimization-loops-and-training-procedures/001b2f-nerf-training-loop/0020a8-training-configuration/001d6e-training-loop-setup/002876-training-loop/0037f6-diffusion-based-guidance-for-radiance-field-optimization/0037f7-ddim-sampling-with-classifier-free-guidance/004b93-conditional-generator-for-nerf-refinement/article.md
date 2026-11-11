# Conditional Generator for NeRF Refinement

Also called: 2D Post-processing Generator, Conditional Generator, GANeRF Generator Stage

## Overview
The Conditional Generator is a 2D post-processing module that refines NeRF renderings by acting as a stochastic denoiser [S1]. It takes a raw NeRF rendering and an auxiliary random vector as input to produce a cleaned image, addressing artifacts that remain after 3D optimization [S1]. Unlike 3D discriminator-based methods that guide the radiance field directly, this method operates in image space, leveraging a StyleGAN2-inspired architecture to enhance perceptual quality without altering the underlying 3D representation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A raw NeRF rendering $R_{\perp_\theta}(r_P)$ (patch or full image) and an auxiliary random vector $z$ [S1].
*   **Output:** A refined, cleaned image patch [S1].

**Data Structures and Architecture:**
The generator $G_\omega$ is a fully convolutional network inspired by StyleGAN2 but adapted as a conditional generator, omitting the mapping network [S1].
*   **Feature Extraction Pyramid:** Consists of six blocks operating at multiple resolutions [S1].
*   **Multi-scale Conditioning:** The input patch is bilinearly downsampled 6 times by a factor of 2 [S1]. Each downscaled version is encoded via a convolutional layer into 32 feature channels [S1]. These features are concatenated to the output of the corresponding generator layer at that scale [S1].
*   **Generative Blocks:** Starting from a $4 \times 4$ down-sampled patch, convolutional blocks extract level-specific features [S1]. Features are upsampled and added with generative noise to the next level [S1].
*   **Normalization and Activation:** Uses normalization layers and leaky ReLU activations [S1].
*   **Noise Injection:** Additive noise is injected after normalization layers, scaled by a learnable per-layer factor [S1].
*   **Skip Connections:** Input and output skip connections are used [S1].
*   **Resolution:** The final output resolution matches the input resolution (typically $256 \times 256$ during training) [S1].

**Training Objective:**
The generator is trained in a regularized adversarial setting alongside a discriminator $D_\psi$ [S1]. The loss function for the generator is:

$$ L_{rgb}^G(\omega | \theta) = \mathbb{E}_{q,n} \|G_\omega(R_{\perp_\theta}(r_P), z) - c_P\|_1 + \lambda_{perc}^G \mathcal{L}_{perc} + \lambda_{adv}^G \max_{\psi \in \Psi} L_{adv}(\omega, \psi | \theta) $$

Where:
*   $R_{\perp_\theta}(r_P)$ is the NeRF rendering of patch $P$ using optimized parameters $\theta$ [S1].
*   $c_P$ is the ground truth color for patch $P$ [S1].
*   $\mathcal{L}_{perc}$ is the perceptual loss [S1].
*   $L_{adv}$ is the adversarial loss [S1].
*   $\lambda_{perc}^G = 1.0$, $\lambda_{rgb}^G = 3.0$, $\lambda_{adv}^G = 5.0$, and $\lambda_{gp}^G = 0.1$ are loss weights [S1].
*   The saddle-point problem is coped with by alternating updates of the generator and discriminator [S1].

**Parameters:**
*   **Initial Training Patch Size:** $256 \times 256$ [S1].
*   **Batch Size:** 8 for the generator [S1].
*   **Discriminator Batch Size:** 32 (the adversarial loss subdivides each $256 \times 256$ patch into 4 non-overlapping $128 \times 128$ patches) [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $2 \times 10^{-3}$ for both generator and discriminator [S1].
*   **Epochs:** 3000 [S1].

**Inference:**
At inference time, the generator runs fully convolutionally on high-resolution images to refine NeRF renderings [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ConditionalGenerator(nn.Module):
    """
    Conditional Generator for NeRF Refinement.
    Inspired by StyleGAN2, adapted as conditional generator.
    """
    def __init__(self, in_channels=3, out_channels=3, latent_dim=512):
        super().__init__()
        # Architecture inspired by StyleGAN2 generator blocks
        # Omitting mapping network; noise z is injected directly
        self.pyramid = StyleGAN2Pyramid(in_channels, out_channels, latent_dim)
        
    def forward(self, nerf_rendering, z):
        """
        Args:
            nerf_rendering: Tensor of shape (B, C, H, W)
            z: Random noise vector of shape (B, latent_dim)
        Returns:
            Refined image of shape (B, C, H, W)
        """
        # Multi-scale conditioning: 6 times bilinearly downsampled by factor of 2
        # This enables multi-scale analysis
        scales = []
        current = nerf_rendering
        for _ in range(6):
            scales.append(current)
            current = F.interpolate(current, scale_factor=0.5, mode='bilinear', align_corners=False)
            
        # Encode downscaled patches into 32 feature channels
        # Concatenate to generator layer output of corresponding scale
        # Note: Implementation details of StyleGAN2Pyramid are assumed standard
        # but adapted to accept these multi-scale features
        refined = self.pyramid(current, z, scales)
        return refined

class StyleGAN2Pyramid(nn.Module):
    def __init__(self, in_ch, out_ch, latent_dim):
        super().__init__()
        # Simplified representation of the 6-block pyramid
        # Starts from 4x4 down-sampled patch
        self.init_conv = nn.Conv2d(in_ch, 512, 3, padding=1)
        self.noise_scale = nn.Parameter(torch.zeros(1))
        # ... other StyleGAN2 blocks (Conv, UpSample, Noise, Add, LeakyReLU, Norm) ...
        # Input/Output skip connections are used
        
    def forward(self, low_res, z, scales):
        # Process through pyramid
        # Inject additive noise after normalization layers, scaled by learnable factor
        # Concatenate encoded multi-scale features
        pass

def train_step(generator, discriminator, nerf_rendering, ground_truth, z, optimizer_g, optimizer_d):
    """
    Alternating update of generator and discriminator.
    """
    # Generator Update
    optimizer_g.zero_grad()
    refined = generator(nerf_rendering, z)
    
    # L1 color loss
    l_rgb = torch.mean(torch.abs(refined - ground_truth))
    
    # Perceptual loss (not detailed in sources, assumed standard LPIPS or VGG)
    l_perc = compute_perceptual_loss(refined, ground_truth)
    
    # Adversarial loss (maximizing discriminator error)
    # L_adv is defined in the adversarial setting
    d_fake = discriminator(refined.detach())
    l_adv = -torch.mean(d_fake) # Simplified GAN loss formulation
    
    loss_g = l_rgb + 1.0 * l_perc + 5.0 * l_adv
    loss_g.backward()
    optimizer_g.step()
    
    # Discriminator Update
    optimizer_d.zero_grad()
    d_real = discriminator(ground_truth)
    d_fake = discriminator(refined)
    
    # Adversarial loss for discriminator (minimizing discriminator error on real, maximizing on fake)
    # Plus gradient penalty lambda_gp = 0.1
    l_adv_d = torch.mean(d_fake) - torch.mean(d_real)
    l_gp = compute_gradient_penalty(discriminator, ground_truth, refined)
    loss_d = l_adv_d + 0.1 * l_gp
    loss_d.backward()
    optimizer_d.step()
```

## Subtopics

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on ScanNet++ indoor scenes (office, lab, apartment) with spatially out-of-distribution test views [S1]. Compared against Mip-NeRF 360, Instant NGP, and Nerfacto variations [S1].
*   **Strengths:** Improves rendering quality in 2D post-processing; handles high-resolution images fully convolutionally at inference; stochastic nature helps with denoising [S1].
*   **Limitations:** Requires additional training of a GAN; operates in 2D, so it cannot fix 3D inconsistencies directly (though it complements 3D discriminator guidance) [S1].
*   **When to use:** When NeRF renderings have perceptual artifacts that 3D optimization alone cannot resolve, and high-resolution refinement is needed [S1].
*   **When not:** When 3D consistency is paramount and 2D post-processing might introduce view-dependent inconsistencies not present in the 3D model [S1].

## Background
The Conditional Generator is part of the GANeRF framework, which leverages discriminators to optimize Neural Radiance Fields [S1]. It was introduced to address artifacts by repairing them in 2D post-processing after 3D optimization [S1]. The architecture is inspired by StyleGAN2 [Karras et al. 2020] and the conditional GAN setting of pix2pix [Isola et al. 2017] [S1].

## Sources
[S1] GANeRF: Leveraging Discriminators to Optimize Neural Radiance Fields, https://doi.org/10.1145/3618402, fetched 2026-10-26
