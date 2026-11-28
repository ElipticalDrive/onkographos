# AR-NeRF Training Configuration

Also called: AR-NeRF Implementation Details, AR-NeRF architecture and training configuration

## Overview
AR-NeRF is a method for unsupervised learning of depth and defocus effects from natural images by combining aperture rendering with neural radiance fields [S1]. It solves the problem of generating photorealistic images with controllable depth-of-field effects without requiring explicit 3D scene reconstruction or ground-truth depth labels [S1]. It differs from standard NeRF variants by integrating a GAN-based training framework (specifically adapted from pi-GAN) to handle the ill-posed nature of single-view synthesis, while explicitly modeling aperture size and focus distance as continuous control parameters [S1].

## How it works
The method trains a generative model that maps a latent code to a radiance field, which is then rendered via volume rendering with aperture control [S1]. The training minimizes a combination of GAN loss, R1 regularization, and an identity regularizer to maintain consistency across viewpoints [S1].

### Inputs and Outputs
*   **Inputs:** A dataset of unposed natural images (e.g., flowers, birds, faces) [S1]. A random latent code $z$ sampled from a standard normal distribution [G].
*   **Outputs:** Synthesized images with controllable defocus strength (aperture size $s$) and focus distance ($f$) [S1].

### Data Structures and Network Layouts
The architecture consists of three main components: a Mapping Network, a Synthesis Network (split into Foreground and Background), and a Discriminator [S1].

1.  **Mapping Network:**
    *   Structure: Multi-Layer Perceptron (MLP).
    *   Hidden Layers: 3 layers.
    *   Units per Layer: 256.
    *   Activation: Leaky ReLU (LReLU) with negative slope 0.2 [S1].
    *   Latent Code Dimension: 256 [S1].
    *   Purpose: Propagates information from the latent code to the synthesis network layers [S1].

2.  **Synthesis Network:**
    *   Structure: MLP using FiLM-SIREN (Fourier Feature SIREN with Feature-wise Linear Modulation) [S1].
    *   **Foreground Synthesis Network:**
        *   Hidden Layers: 8 layers.
        *   Units per Layer: 128 [S1].
        *   Note: Reduced from 256 (used in original pi-GAN) to allow more points per ray, improving performance without significant quality loss [S1].
    *   **Background Synthesis Network:**
        *   Hidden Layers: 8 layers.
        *   Units per Layer: 64 [S1].
        *   Rationale: Fewer parameters are used under the assumption that the background is simpler than the foreground [S1].

3.  **Discriminator:**
    *   Structure: Convolutional network with residual blocks [S1].
    *   **For FFHQ (Faces):** Includes CoordConv layers followed by residual blocks [S1].
    *   **For Oxford Flowers and CUB-200-2011:** Same as FFHQ but **without** CoordConv layers [S1]. CoordConv was found to have negative effects on these datasets likely due to the lack of strict alignment (unlike facial landmarks in FFHQ) [S1].

### Training Steps and Parameters
The training process uses the non-saturating GAN loss with real gradient penalty (R1) regularization [S1].

1.  **Optimization Setup:**
    *   Optimizer: Adam [S1].
    *   Learning Rate: 0.00006 [S1].
    *   Total Iterations: 200,000 [S1].
    *   R1 Regularization Weight: 0.2 [S1].
    *   Identity Regularizer Weight: 15 [S1]. The identity regularizer ensures that the generated content remains consistent (preserves identity) across different viewpoints [S1].

2.  **Ray Sampling and Rendering:**
    *   The synthesis network outputs density and color [S1].
    *   Ray sampling involves querying the network at points along the ray [S1].
    *   The number of points per ray is increased to improve performance, facilitated by the reduced unit count (128) in the foreground network [S1].
    *   Defocus parameters (aperture size $s$, focus distance $f$) are integrated into the rendering process to simulate camera optics [S1].

3.  **Loss Function:**
    The total loss $L$ is a weighted sum:
    $$ L = L_{GAN} + \lambda_{R1} L_{R1} + \lambda_{ID} L_{ID} $$
    Where:
    *   $L_{GAN}$ is the non-saturating GAN loss [S1].
    *   $L_{R1}$ is the R1 gradient penalty regularization [S1].
    *   $\lambda_{R1} = 0.2$ [S1].
    *   $L_{ID}$ is the identity regularizer loss [S1].
    *   $\lambda_{ID} = 15$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn as nn
import torch.optim as optim

# Helper: Multi-Layer Perceptron
class MLP(nn.Module):
    def __init__(self, hidden_layers, units, activation):
        super().__init__()
        self.layers = nn.ModuleList([nn.Linear(units, units) for _ in range(hidden_layers)])
        self.activation = activation

    def forward(self, x):
        for layer in self.layers:
            x = layer(x)
            if self.activation == 'LReLU':
                x = torch.nn.functional.leaky_relu(x, negative_slope=0.2)
        return x

# Helper: Sample latent code
def sample_from_normal_distribution(batch_size, latent_dim=256):
    return torch.randn(batch_size, latent_dim)

# Helper: AR-NeRF Rendering (simplified volume rendering)
def render_arnerf(z, foreground_net, background_net, mapping_net, num_points=10):
    z_mapped = mapping_net(z)
    # Query networks along ray; returns density and color tensors
    return foreground_net(z_mapped), background_net(z_mapped)

# Helper: Non-saturating GAN loss
def compute_gan_loss(real_images, generated_images, discriminator):
    real_logits = discriminator(real_images)
    fake_logits = discriminator(generated_images)
    loss = torch.mean(torch.nn.functional.softplus(-fake_logits))
    return loss

# Helper: R1 Gradient Penalty
def compute_r1_gradient_penalty(real_images, discriminator):
    real_images.requires_grad_(True)
    real_logits = discriminator(real_images)
    gradients = torch.autograd.grad(outputs=real_logits.sum(), inputs=real_images, create_graph=True)[0]
    penalty = torch.mean(torch.square(gradients))
    return penalty

# Helper: Identity Regularizer
def compute_identity_loss(generated_images, mapping_net):
    # Ensures consistency across viewpoints; formula not specified in sources [G]
    return torch.mean(torch.square(generated_images))

# Helper: Discriminator
class Discriminator(nn.Module):
    def __init__(self, with_coordconv):
        super().__init__()
        self.with_coordconv = with_coordconv
        # CoordConv layers followed by residual blocks for FFHQ; removed for Flowers/CUB [S1]
        self.conv = nn.Conv2d(3, 64, 4, stride=2, padding=1)
        self.resnet = nn.Sequential(nn.Conv2d(64, 128, 3, padding=1), nn.ReLU())

    def forward(self, x):
        x = self.conv(x)
        x = self.resnet(x)
        return torch.mean(x, dim=(2, 3))

def train_arnerf(dataset, num_iterations=200000, batch_size=32):
    # 1. Initialize Networks
    mapping_net = MLP(hidden_layers=3, units=256, activation='LReLU')
    foreground_net = MLP(hidden_layers=8, units=128, activation='FiLM-SIREN')
    background_net = MLP(hidden_layers=8, units=64, activation='FiLM-SIREN')
    
    # Discriminator setup depends on dataset
    discriminator = Discriminator(with_coordconv=True) # Default for FFHQ; set False for Flowers/CUB [S1]
        
    # 2. Initialize Optimizer
    optimizer = optim.Adam(list(mapping_net.parameters()) + list(foreground_net.parameters()) + 
                           list(background_net.parameters()) + list(discriminator.parameters()), lr=0.00006)
    
    # 3. Hyperparameters
    lambda_r1 = 0.2
    lambda_id = 15
    
    # 4. Training Loop
    for iteration in range(num_iterations):
        real_images = dataset.next_batch(batch_size)
        
        z = sample_from_normal_distribution(batch_size)
        
        generated_images = render_arnerf(z, foreground_net, background_net, mapping_net)
        
        gan_loss = compute_gan_loss(real_images, generated_images, discriminator)
        r1_loss = compute_r1_gradient_penalty(real_images, discriminator)
        id_loss = compute_identity_loss(generated_images, mapping_net)
        
        total_loss = gan_loss + (lambda_r1 * r1_loss) + (lambda_id * id_loss)
        
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
```

### Edge Cases and Pitfalls
*   **CoordConv Usage:** Using CoordConv layers in the discriminator for non-aligned objects (flowers, birds) degrades performance [S1].
*   **Ray Density:** Processing time and memory increase with the number of rays [S1]. While 10 rays were found sufficient for comparable performance to stratified sampling with fewer rays, higher resolutions may require more points per ray [S1].
*   **Unit Reduction:** Reducing synthesis network units from 256 to 128 is critical for allowing more points per ray, which improves performance [S1].

## Subtopics

### AR-NeRF-R
A defocus renderer trained on AR-NeRF-generated images that allows continuous and intuitive manipulation of defocus strength and focus distance given an input image [S1]. Unlike AR-GAN-DR, where defocus parameters are uniquely determined by the input image, AR-NeRF-R allows interactive adjustment via auxiliary inputs [S1].
*   **Definition:** A conditional U-Net that takes an image and auxiliary parameters (aperture size $s$, focus distance $f$) to output a defocused image [S1].
*   **Training Steps:**
    1.  Generate training images at 128×128 resolution using AR-NeRF by increasing input point density [S1].
    2.  Train a conditional U-Net on these images [S1].
    3.  Feed aperture size $s$ and focus distance $f$ as auxiliary information to the U-Net [S1].
    4.  Output defocused images by varying $s$ and $f$ continuously [S1].
*   **Layout:** U-Net architecture with two auxiliary scalar inputs ($s, f$) concatenated or injected as conditioning signals [S1].
*   **Usage:** Trained on Oxford Flowers dataset using 128×128 images generated by AR-NeRF; evaluated on iPhone2DSLR Flower dataset [S1].

### RegNeRF Training Configuration
An adaptation of RegNeRF using default DTU dataset settings with specific adjustments for compatibility with the AR-NeRF pipeline [S2].
*   **Definition:** A baseline configuration for RegNeRF used in comparative experiments [S2].
*   **Steps:**
    1.  Use default settings provided by authors for the DTU dataset [S2].
    2.  Adjust the near and far planes and scene scaling [S2].
    3.  Disable the loss from the appearance regularizer because the model is not available in this context [S2].

## Operating characteristics
*   **Performance:** AR-NeRF achieves competitive FID and KID scores compared to baseline GANs (AR-GAN, AR-GAN++, RGBD-GAN) on Oxford Flowers, CUB-200-2011, and FFHQ datasets [S1].
*   **Memory/Time:** Processing time and memory usage scale with the number of rays and points per ray. The reduction in network units (128 vs 256) helps mitigate memory constraints, allowing for higher point density [S1].
*   **Strengths:** Unsupervised learning of depth and defocus; continuous control of defocus parameters; high-quality synthesis on natural images [S1].
*   **Limitations:** Requires careful tuning of discriminator architecture (CoordConv) based on dataset alignment; performance may degrade if ray density is insufficient for higher resolutions [S1].

## Background
AR-NeRF was introduced in 2022 by the authors of the paper "AR-NeRF: Unsupervised Learning of Depth and Defocus Effects from Natural Images with Aperture Rendering Neural Radiance Fields" [S1]. It builds upon the pi-GAN framework, adapting it for unsupervised depth and defocus learning [S1]. The method was applied to three datasets: Oxford Flowers (8,189 images, 102 categories), CUB-200-2011 (11,788 images, 200 bird species), and FFHQ (70,000 face images) [S1].

## Sources
[S1] AR-NeRF: Unsupervised Learning of Depth and Defocus Effects from Natural Images with Aperture Rendering Neural Radiance Fields, https://arxiv.org/abs/2206.06100v1, fetched 2026-10-26
[S2] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
