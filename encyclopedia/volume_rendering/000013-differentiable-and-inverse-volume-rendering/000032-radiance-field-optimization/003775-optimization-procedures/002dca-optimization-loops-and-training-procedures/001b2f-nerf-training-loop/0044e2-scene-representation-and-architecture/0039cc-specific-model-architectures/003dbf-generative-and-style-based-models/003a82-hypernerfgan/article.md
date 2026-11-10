# HyperNeRFGAN

Also called: Hypernetwork NeRF GAN, HyperNeRF-GAN

## Overview
HyperNeRFGAN is a generative adversarial network (GAN) architecture that uses a hypernetwork to produce the weights of a simplified NeRF target network, enabling 3D-aware image generation without requiring camera position information during training [S1, unofficial]. Unlike standard NeRF which uses a 5D input $(x, d)$, HyperNeRFGAN uses a 3D input $(x)$ because training images lack view-dependent characteristics like reflections [S1, unofficial]. It sets itself apart from siblings like [Latent-NeRF](9110-latent-nerf.md) by avoiding the complexity of NeRF conditioning mechanisms (used in GRAF and $\pi$-GAN) while maintaining 3D awareness through direct weight generation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A noise vector $\mathbf{z}$ sampled from a Gaussian distribution $P_{noise}$ [S1, unofficial].
*   **Output:** A rendered 2D image $C(\mathbf{r})$ from a generated 3D representation [S1, unofficial].

**Data Structures:**
The model consists of two main components:
1.  **Generator ($G$):** A hypernetwork based on the StyleGAN2 backbone. It takes $\mathbf{z}$ and outputs the weights $\theta$ for the target NeRF model $F_\theta$ [S1, unofficial].
2.  **Target NeRF ($F_\theta$):** A single Multi-Layer Perceptron (MLP) that maps 3D spatial positions to color and density. It omits viewing direction as input [S1, unofficial].
    *   **Architecture:** The MLP uses Factorized Multiplicative Modulation (FMM) layers instead of standard linear layers [S1, unofficial].
    *   **FMM Layer Formula:**
        $$
        \mathbf{y} = \mathbf{W} \odot (\mathbf{A} \times \mathbf{B}) \cdot \mathbf{x}_{in} + \mathbf{b} = \tilde{\mathbf{W}} \cdot \mathbf{x}_{in} + \mathbf{b}
        $$
        where $\mathbf{W}$ and $\mathbf{b}$ are shared matrices across 3D representations, and $\mathbf{A}$ and $\mathbf{B}$ are modulation matrices created by $G$ with dimensions $n_{out} \times k$ and $k \times n_{in}$ respectively [S1, unofficial]. The parameter $k$ controls the rank; the source uses $k=10$ [S1, unofficial].
    *   **Simplifications:** To reduce computational expense, the model rejects the larger "fine" network used in standard NeRF and employs only the smaller "coarse" network [S1, unofficial]. The number of channels in each hidden layer is reduced from 256 to 128, and the number of layers is reduced from 8 to 4 in some experiments [S1, unofficial].
3.  **Discriminator ($D$):** A standard 2D discriminator that evaluates the rendered 2D images [S1, unofficial].

**Numbered Steps:**
1.  Sample a noise vector $\mathbf{z}$ from the base Gaussian distribution $P_{noise}$ [S1, unofficial].
2.  Transform $\mathbf{z}$ using the generator $G$ to obtain the weights $\theta$ of the target NeRF model $F_\theta$ [S1, unofficial].
3.  Instantiate the target model $F_\theta$ with weights $\theta$. This model maps spatial location $\mathbf{x}=(x,y,z)$ to emitted color $\mathbf{c}=\mathbf{c}(\mathbf{x})$ and volume density $\sigma=\sigma(\mathbf{x})$ [S1, unofficial].
4.  Render 2D images from a variety of perspectives using $F_\theta$ and the volumetric rendering equation [S1, unofficial].
5.  Train the generator $G$ and discriminator $D$ using the minimax game:
    $$
    \min_G \max_D V(D, G)
    $$
    where
    $$
    V = \mathbb{E}_{\mathbf{x} \sim P_{data}}[\log D(\mathbf{x})] + \mathbb{E}_{\mathbf{z} \sim P_{noise}}[\log(1 - D(G(\mathbf{z})))]
    $$
    [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FMMLayer(nn.Module):
    """Factorized Multiplicative Modulation Layer"""
    def __init__(self, n_in, n_out, k=10):
        super().__init__()
        self.n_in = n_in
        self.n_out = n_out
        self.k = k
        # Shared weights
        self.W = nn.Parameter(torch.randn(n_out, n_in))
        self.b = nn.Parameter(torch.zeros(n_out))
        # Modulation matrices dimensions: A is n_out x k, B is k x n_in
        # These are NOT parameters of the layer itself, but outputs of the Generator G
        # For the purpose of the NeRF forward pass, they are inputs.
        
    def forward(self, x_in, A, B):
        # A: (n_out, k), B: (k, n_in)
        # Compute modulation: W_tilde = W * (A @ B)
        # Note: Broadcasting rules apply. W is (n_out, n_in). A@B is (n_out, n_in).
        W_tilde = self.W * (A @ B)
        return F.linear(x_in, W_tilde, self.b)

class HyperNeRFGAN:
    def __init__(self, generator, discriminator, n_layers=4, channels=128, k=10):
        self.G = generator
        self.D = discriminator
        self.n_layers = n_layers
        self.channels = channels
        self.k = k
        # The target NeRF structure is defined by the generator's output dimensions
        # but logically consists of FMM layers.
        
    def get_nerf_weights(self, z):
        """
        Generator G takes noise z and outputs modulation matrices A and B
        for each layer of the NeRF.
        Returns: List of tuples (A, B) for each layer.
        """
        # G is a StyleGAN2-based hypernetwork.
        # Implementation detail: G outputs the modulation parameters.
        # The exact internal structure of G is not specified in the source,
        # but it maps z -> {A_l, B_l}_{l=1}^{n_layers}
        return self.G(z)

    def render_ray(self, ray_origin, ray_direction, num_samples, nerf_weights):
        """
        Renders a single ray using the target NeRF F_theta.
        F_theta: x -> (c, sigma)
        """
        # Sample points along the ray
        # t = linspace(t_near, t_far, num_samples)
        # x = ray_origin + t * ray_direction
        # Note: The source does not specify t_near/t_far or sampling strategy.
        # We assume standard uniform sampling for the coarse network.
        
        # For each point x_i:
        #   Apply positional encoding gamma(x_i) [Parent]
        #   Pass through NeRF layers using weights from nerf_weights
        #   Output sigma_i, c_i
        
        # Volumetric rendering
        # alpha_i = 1 - exp(-sigma_i * delta_i)
        # T_i = product(1 - alpha_j) for j < i
        # C(r) = sum(T_i * alpha_i * c_i)
        
        # Since the source does not provide the exact NeRF forward pass pseudocode,
        # we describe the logical flow.
        # The NeRF is a sequence of FMM layers.
        # Input to first layer: gamma(x)
        # Output of last layer: (sigma, color)
        
        # Placeholder for NeRF forward pass logic
        # x_encoded = positional_encoding(x)
        # h = x_encoded
        # for l in range(n_layers):
        #     A, B = nerf_weights[l]
        #     h = FMMLayer(channels, channels, k).forward(h, A, B)
        # sigma, color = head(h)
        
        # Standard volumetric integration
        # ...
        pass

    def train_step(self, real_images, z_batch):
        # 1. Generate NeRF weights
        nerf_weights = self.get_nerf_weights(z_batch)
        
        # 2. Render fake images using the generated NeRF
        # This requires defining camera poses for rendering.
        # The source states "render 2D images from a variety of perspectives".
        # It does not specify how perspectives are chosen if camera positions are unknown.
        # [G] Assume random camera poses are sampled for training.
        fake_images = self.render_batch(neural_weights, camera_poses)
        
        # 3. Discriminator loss
        d_real = self.D(real_images)
        d_fake = self.D(fake_images.detach())
        loss_D = -torch.mean(torch.log(d_real + eps) + torch.log(1 - d_fake + eps))
        
        # 4. Generator loss
        d_fake_gen = self.D(fake_images)
        loss_G = -torch.mean(torch.log(d_fake_gen + eps))
        
        # 5. Update weights
        # ...
        return loss_D, loss_G
```

**Parameters:**
*   **Hidden layer channels:** 128 (reduced from 256) [S1, unofficial].
*   **Network depth:** 4 layers (reduced from 8) in some experiments [S1, unofficial].
*   **Rank $k$:** 10 [S1, unofficial].
*   **Input noise:** Gaussian distribution [S1, unofficial].

**Edge cases and pitfalls:**
*   **Viewing Directions:** Standard NeRF requires viewing directions, which are often unavailable. HyperNeRFGAN solves this by removing viewing directions from the NeRF input [S1, unofficial].
*   **Fine Network:** Standard NeRF uses a 'fine' network for multi-scale rendering; HyperNeRFGAN rejects the larger 'fine' network and employs only the smaller 'coarse' network to reduce training cost [S1, unofficial].
*   **Rank $k$:** Higher values of $k$ increase expressiveness but also increase memory usage [S1, unofficial].

## Subtopics

### Latent-NeRF
Uses a latent code to condition the NeRF, allowing for view-dependent effects and explicit control over scene attributes. Choose when view-dependent rendering or explicit latent manipulation is required. [Latent-NeRF](9110-latent-nerf.md)

## Operating characteristics
*   **Time and Memory Complexity:** Reduced compared to standard NeRF due to the use of only the coarse network, reduced channel count (128), and reduced depth (4 layers) [S1, unofficial].
*   **Measured Performance:**
    *   **ShapeNet Car:** FID 29.6 (vs Point2NeRF 82.1) [S1, unofficial].
    *   **CARLA Dataset:** FID 20.5, KID 0.78, IS 4.20 (outperforming HoloGAN FID 67.5, GRAF FID 41.7, $\pi$-GAN FID 29.2) [S1, unofficial].
    *   **CelebA Dataset:** FID 15.04, KID 0.66, IS 2.63 (comparable to $\pi$-GAN FID 14.7, KID 0.39, IS 2.62) [S1, unofficial].
*   **Strengths:** Superior performance on datasets where camera position estimation is challenging (e.g., medical data, single-view datasets) [S1, unofficial]. Simpler than existing SOTA alternatives [S1, unofficial].
*   **Limitations:** Cannot model view-dependent effects (like reflections) because viewing directions are omitted [S1, unofficial].
*   **When to use:** When camera positions are unknown or hard to estimate, and the scene lacks strong view-dependent characteristics [S1, unofficial].
*   **When not to use:** When view-dependent rendering is required [S1, unofficial].

## Background
HyperNeRFGAN was introduced in a paper titled "HyperNeRFGAN: Hypernetwork approach to 3D NeRF GAN" [S1, unofficial]. It builds on the hypernetwork paradigm (Ha et al., 2016) and the StyleGAN2 architecture [S1, unofficial]. It is used for 3D object generation on ShapeNet (car, plane, chair) and CARLA datasets, and for medical applications on Digitally Reconstructed Radiographs (DRR) of chests and knees [S1, unofficial].

## Sources
[S1] HyperNeRFGAN: Hypernetwork approach to 3D NeRF GAN, https://doi.org/10.48550/arxiv.2301.11631, fetched 2026-10-29 (unofficial)
