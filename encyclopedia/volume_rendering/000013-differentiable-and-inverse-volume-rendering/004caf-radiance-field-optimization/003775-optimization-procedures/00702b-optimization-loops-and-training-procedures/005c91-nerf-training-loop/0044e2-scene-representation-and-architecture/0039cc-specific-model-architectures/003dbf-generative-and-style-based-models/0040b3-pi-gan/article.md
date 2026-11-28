# pi-GAN

Also called: 3D Generative Adversarial Models (GANs), GRAF / pi-GAN, Periodic Implicit Generative Adversarial Networks

## Overview
pi-GAN is a generative adversarial network (GAN) that learns an unconditional 3D-aware image synthesis model by embedding a conditional neural radiance field (NeRF) into the generator [S1, S3]. Unlike standard 2D GANs, it produces a 5D spatio-angular radiance field parameterized by a SIREN network, conditioned on a latent noise vector via Feature-wise Linear Modulation (FiLM) [S3]. It distinguishes itself from its sibling GRAF by replacing concatenation-based conditioning with FiLM, using a SIREN backbone instead of ReLU MLPs, and employing a global discriminator rather than a patch-based one to avoid local overfitting [S1, S3].

## How it works

### Inputs and Outputs
*   **Input:** A latent noise vector $\mathbf{z} \sim p_z$ (typically standard normal) and a camera pose $\boldsymbol{\xi}$ (defining the ray origin $\mathbf{o}$ and direction $\mathbf{d}$) [S3].
*   **Output:** A 2D image $I$ rendered from the camera pose $\boldsymbol{\xi}$ [S3].

### Data Structures and Layouts

**1. Mapping Network ($M$)**
*   **Type:** Multi-Layer Perceptron (MLP) with ReLU activations [S3].
*   **Input:** Latent vector $\mathbf{z} \in \mathbb{R}^{d_z}$ [S3].
*   **Output:** A set of frequency parameters $\{\gamma_i\}_{i=0}^{n-1}$ and phase shift parameters $\{\beta_i\}_{i=0}^{n-1}$, one pair per layer of the SIREN backbone [S3].
*   **Layout:** The output is a flat vector of size $2 \times n \times N_{hidden}$, where $n$ is the number of SIREN layers and $N_{hidden}$ is the hidden dimension of the SIREN [S3].

**2. SIREN Backbone ($\Phi$)**
*   **Type:** MLP with periodic sine activations [S3].
*   **Input:** 3D spatial coordinate $\mathbf{x} \in \mathbb{R}^3$ [S3].
*   **Layers:** $n$ hidden layers. Each layer $i$ applies an affine transformation followed by a sine non-linearity [S3].
*   **Parameters:**
    *   $\mathbf{W}_i \in \mathbb{R}^{N_i \times M_i}$: Weight matrix for layer $i$.
    *   $\mathbf{b}_i \in \mathbb{R}^{N_i}$: Bias vector for layer $i$.
    *   $\gamma_i \in \mathbb{R}^{N_i}$: Frequency scaling factor for layer $i$ (from Mapping Network).
    *   $\beta_i \in \mathbb{R}^{N_i}$: Phase shift for layer $i$ (from Mapping Network).
*   **Activation Function for layer $i$:**
    $$ \phi_i(\mathbf{x}_i) = \sin(\gamma_i \cdot (\mathbf{W}_i \mathbf{x}_i + \mathbf{b}_i) + \beta_i) $$
    where $\cdot$ denotes element-wise multiplication [S3].

**3. Density Head**
*   **Input:** Output of the final SIREN layer $\Phi(\mathbf{x}) \in \mathbb{R}^{N_n}$ [S3].
*   **Parameters:** $\mathbf{W}_\sigma \in \mathbb{R}^{1 \times N_n}$, $\mathbf{b}_\sigma \in \mathbb{R}$ [S3].
*   **Output:** Density $\sigma(\mathbf{x}) \in \mathbb{R}$ [S3].

**4. Color Head**
*   **Input:** Concatenation of the SIREN output $\Phi(\mathbf{x})$ and the viewing direction $\mathbf{d}$ [S3].
*   **Parameters:** $\mathbf{W}_c \in \mathbb{R}^{3 \times (N_n + 3)}$, $\mathbf{b}_c \in \mathbb{R}^3$ [S3].
*   **Activation:** $\phi_c$ is typically a linear or ReLU layer (source specifies $\phi_c$ generally, but standard NeRF uses ReLU; pi-GAN paper implies a final linear projection for RGB) [S3].
*   **Output:** Color $\mathbf{c}(\mathbf{x}, \mathbf{d}) \in \mathbb{R}^3$ [S3].

**5. Discriminator**
*   **Type:** Traditional convolutional neural network (CNN) [S3].
*   **Input:** 2D rendered image [S3].
*   **Output:** Scalar logits for real/fake classification [S3].

### Numbered Steps

1.  **Latent Sampling:** Sample a noise vector $\mathbf{z}$ from the prior distribution $p_z$ [S3].
2.  **Mapping:** Pass $\mathbf{z}$ through the Mapping Network $M$ to obtain frequencies $\{\gamma_i\}$ and phase shifts $\{\beta_i\}$ [S3].
3.  **Radiance Field Construction:** Define the implicit radiance field using the SIREN backbone $\Phi$ conditioned by $\{\gamma_i, \beta_i\}$ [S3].
    *   Compute the backbone output for a 3D point $\mathbf{x}$:
        $$ \Phi(\mathbf{x}) = \phi_{n-1}(\dots \phi_0(\mathbf{x}) \dots) $$
        where $\phi_i(\mathbf{x}_i) = \sin(\gamma_i \cdot (\mathbf{W}_i \mathbf{x}_i + \mathbf{b}_i) + \beta_i)$ [S3].
4.  **Density Prediction:** Compute density $\sigma(\mathbf{x})$ using the density head:
    $$ \sigma(\mathbf{x}) = \mathbf{W}_\sigma \Phi(\mathbf{x}) + \mathbf{b}_\sigma $$
    [S3].
5.  **Color Prediction:** Compute view-dependent color $\mathbf{c}(\mathbf{x}, \mathbf{d})$ using the color head:
    $$ \mathbf{c}(\mathbf{x}, \mathbf{d}) = \mathbf{W}_c \phi_c \left( [\Phi(\mathbf{x}), \mathbf{d}]^T \right) + \mathbf{b}_c $$
    where $[\cdot, \cdot]^T$ denotes concatenation [S3].
6.  **Volume Rendering:** Cast rays from camera origin $\mathbf{o}$ with direction $\mathbf{d}$ defined by pose $\boldsymbol{\xi}$. Sample points $\mathbf{x}_i$ along the ray. Compute pixel color $C(\mathbf{r})$ using the standard volumetric rendering equation:
    $$ C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $\delta_i$ is the distance between samples [S3, G].
7.  **Adversarial Training:**
    *   **Generator Loss:** The rendered image is passed to the Discriminator. The generator is trained to minimize the adversarial loss (e.g., non-saturating GAN loss) by fooling the discriminator [S3].
    *   **Discriminator Loss:** The discriminator is trained to maximize the likelihood of correctly classifying real images (from the dataset) and generated images (from the generator) [S3].
8.  **Progressive Growing:** During training, the resolution of the rendered images and the discriminator is progressively increased to accelerate convergence and stabilize training [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MappingNetwork(nn.Module):
    """
    Maps latent vector z to SIREN parameters (frequencies and phase shifts).
    """
    def __init__(self, z_dim, n_layers, hidden_dim):
        super().__init__()
        self.n_layers = n_layers
        self.hidden_dim = hidden_dim
        # Simple ReLU MLP as described in [S3]
        layers = []
        layers.append(nn.Linear(z_dim, hidden_dim))
        layers.append(nn.ReLU())
        for _ in range(n_layers - 1):
            layers.append(nn.Linear(hidden_dim, hidden_dim))
            layers.append(nn.ReLU())
        self.mlp = nn.Sequential(*layers)
        # Output 2 * n_layers * hidden_dim values
        self.param_head = nn.Linear(hidden_dim, 2 * n_layers * hidden_dim)

    def forward(self, z):
        h = self.mlp(z)
        params = self.param_head(h)
        # Reshape to (n_layers, 2, hidden_dim) -> (n_layers, hidden_dim) for gamma and beta
        params = params.view(self.n_layers, 2, self.hidden_dim)
        gammas = params[:, 0, :]
        betas = params[:, 1, :]
        return gammas, betas

class SIRENLayer(nn.Module):
    def __init__(self, in_features, out_features, gamma, beta):
        super().__init__()
        self.linear = nn.Linear(in_features, out_features, bias=True)
        # Initialize weights for sine activation stability [G]
        self.linear.weight.data.uniform_(-1 / in_features, 1 / in_features)
        self.linear.bias.data.uniform_(-1 / in_features, 1 / in_features)
        self.gamma = gamma
        self.beta = beta

    def forward(self, x):
        # phi_i(x_i) = sin(gamma_i * (W_i * x_i + b_i) + beta_i)
        affine = self.linear(x)
        return torch.sin(self.gamma * affine + self.beta)

class SIRENBackbone(nn.Module):
    def __init__(self, in_dim, n_layers, hidden_dim, gammas, betas):
        super().__init__()
        self.layers = nn.ModuleList()
        for i in range(n_layers):
            in_f = in_dim if i == 0 else hidden_dim
            out_f = hidden_dim
            # gammas and betas are tensors of shape (n_layers, hidden_dim)
            self.layers.append(SIRENLayer(in_f, out_f, gammas[i], betas[i]))

    def forward(self, x):
        h = x
        for layer in self.layers:
            h = layer(h)
        return h

class PiGANGenerator(nn.Module):
    def __init__(self, z_dim, n_layers, hidden_dim, n_samples):
        super().__init__()
        self.mapping = MappingNetwork(z_dim, n_layers, hidden_dim)
        self.n_layers = n_layers
        self.hidden_dim = hidden_dim
        self.n_samples = n_samples
        # Density head
        self.density_head = nn.Linear(hidden_dim, 1)
        # Color head
        self.color_head = nn.Linear(hidden_dim + 3, 3) # [Phi(x), d] -> RGB

    def forward(self, z, camera_poses):
        """
        z: (B, z_dim)
        camera_poses: (B, 6) or similar structure defining rays [G]
        Returns: rendered images (B, H, W, 3)
        """
        # 1. Map z to SIREN parameters
        gammas, betas = self.mapping(z) # (B, n_layers, hidden_dim)

        # 2. Sample points along rays [G]
        # Assuming camera_poses provides ray origins o and directions d
        # For simplicity, assume we have sampled points x_samples (B, N, 3)
        # and directions d_samples (B, N, 3) corresponding to these points
        # In a real implementation, this involves ray casting logic
        x_samples, d_samples = self.sample_rays(camera_poses)

        # 3. Compute SIREN output for all points
        # Expand gammas/betas to match batch and sample dimensions
        # gammas: (B, n_layers, hidden_dim) -> (B, N, n_layers, hidden_dim)
        gammas_exp = gammas.unsqueeze(1).expand(-1, self.n_samples, -1, -1)
        betas_exp = betas.unsqueeze(1).expand(-1, self.n_samples, -1, -1)
        
        # We need to process each layer sequentially for the SIREN
        # However, to vectorize, we can compute the full forward pass
        # Note: The SIREN backbone definition in [S3] is sequential composition.
        # We will implement a helper that processes the batch of points through the SIREN
        
        phi_output = self.compute_siren_output(x_samples, gammas_exp, betas_exp)

        # 4. Predict Density
        # phi_output shape: (B, N, hidden_dim)
        sigma = self.density_head(phi_output).squeeze(-1) # (B, N)

        # 5. Predict Color
        # Concatenate phi_output and d_samples
        # d_samples shape: (B, N, 3)
        color_input = torch.cat([phi_output, d_samples], dim=-1) # (B, N, hidden_dim + 3)
        color = self.color_head(color_input) # (B, N, 3)

        # 6. Volume Rendering
        images = self.volume_render(x_samples, sigma, color, camera_poses)
        return images

    def compute_siren_output(self, x, gammas, betas):
        """
        Computes the output of the SIREN backbone for a batch of points.
        x: (B, N, 3)
        gammas: (B, n_layers, hidden_dim)
        betas: (B, n_layers, hidden_dim)
        Returns: (B, N, hidden_dim)
        """
        B, N, _ = x.shape
        n_layers = self.n_layers
        hidden_dim = self.hidden_dim
        
        h = x # (B, N, 3)
        
        for i in range(n_layers):
            W = self.mapping.layers[i].linear.weight # (hidden_dim, in_dim)
            b = self.mapping.layers[i].linear.bias   # (hidden_dim,)
            gamma_i = gammas[:, i, :]                # (B, hidden_dim)
            beta_i = betas[:, i, :]                  # (B, hidden_dim)
            
            # Affine transform: W @ x.T + b
            # h: (B, N, in_dim) -> (B, N, hidden_dim)
            h = torch.matmul(h, W.T) + b
            
            # Sine activation: sin(gamma * h + beta)
            # gamma and beta are broadcasted over N
            h = torch.sin(gamma_i.unsqueeze(1) * h + beta_i.unsqueeze(1))
            
        return h

    def sample_rays(self, camera_poses):
        # Placeholder for ray sampling logic [G]
        # Returns x_samples (B, N, 3) and d_samples (B, N, 3)
        pass

    def volume_render(self, x_samples, sigma, color, camera_poses):
        # Placeholder for volumetric rendering logic [G]
        # Uses the formula: C(r) = sum T_i alpha_i c_i
        pass

class Discriminator(nn.Module):
    def __init__(self):
        super().__init__()
        # Traditional CNN discriminator [S3]
        self.conv = nn.Sequential(
            nn.Conv2d(3, 64, 4, stride=2, padding=1),
            nn.LeakyReLU(0.2, inplace=True),
            # ... more layers
            nn.Conv2d(64, 1, 4, stride=1, padding=0),
            nn.Sigmoid() # Or linear logits depending on loss function
        )

    def forward(self, x):
        return self.conv(x)
```

### Parameters and Typical Values
*   **Latent Dimension ($d_z$):** Not given in the sources.
*   **SIREN Layers ($n$):** Not given in the sources.
*   **Hidden Dimension ($N_{hidden}$):** Not given in the sources.
*   **Number of Samples ($N$):** Not given in the sources.
*   **Learning Rate:** Not given in the sources.

### Edge Cases and Pitfalls
*   **Local Overfitting:** SIREN networks are prone to local overfitting to the last batch if sufficient coverage of the space is not maintained [S3]. This is the primary reason pi-GAN avoids the patch-based discriminator used in GRAF, opting instead for a global convolutional discriminator [S3].
*   **Spectral Bias:** While SIREN mitigates spectral bias compared to ReLU MLPs, proper initialization of weights (uniform distribution scaled by input dimension) is critical for stable training [S3, G].

## Subtopics

### GRAF
The predecessor to pi-GAN. Uses concatenation-based conditioning of a ReLU MLP. pi-GAN improves upon it with FiLM conditioning and SIREN. [GRAF](8658-graf.md)

### StyleNeRF
Uses style-based generation but focuses on style transfer rather than unconditional 3D-aware synthesis from unposed images. [StyleNeRF](8659-stylenerf.md)

### HyperNeRFGAN
Uses hypernetworks to generate NeRF weights, differing from pi-GAN's direct FiLM conditioning of a SIREN. [HyperNeRFGAN](8765-hypernerfgan.md)

### Latent-NeRF
Infers latent codes from observations for conditional reconstruction, whereas pi-GAN is unconditional. [Latent-NeRF](9110-latent-nerf.md)

### GIRAFFE
Extends the generative NeRF framework to compositional scenes with multiple objects, using a more complex feature field representation. [GIRAFFE](11360-giraffe.md)

### Generative Latent Optimization
Jointly optimizes latent codes with the scene model for reconstruction, not generative synthesis. [Generative Latent Optimization](11963-generative-latent-optimization.md)

### Adversarial Training (GAN)
The general framework used by pi-GAN. [Adversarial Training (GAN)](14030-adversarial-training-gan.md)

## Operating characteristics

*   **Time Complexity:** Training is computationally expensive due to the 3D nature of the representation and the need for volume rendering per image. Progressive growing helps mitigate this [S3].
*   **Memory Complexity:** High, as the SIREN network must be evaluated for many samples along each ray during rendering. The global discriminator also consumes significant memory at high resolutions [S3].
*   **Measured Performance:** Achieved state-of-the-art results on CelebA, Cats, and CARLA datasets at the time of publication [S3]. Outperformed GRAF [S1].
*   **Strengths:**
    *   High-quality, photorealistic 3D-aware image synthesis [S3].
    *   View-consistent rendering from arbitrary camera poses [S3].
    *   Extractable 3D proxy shapes via marching cubes [S3].
*   **Limitations:**
    *   Prone to local overfitting if training coverage is insufficient [S3].
    *   Computational cost of 3D GANs is higher than 2D GANs [S3].
    *   Proxy shapes may not be as high quality as explicit supervision methods [S3].
*   **When to Use:** When high-quality 3D-aware image synthesis is required from unposed 2D images, and computational resources allow for 3D GAN training [S3].
*   **When Not to Use:** When explicit 3D geometry is required with high fidelity (consider surface representations) or when computational resources are limited [S3].

## Background

*   **Origin:** pi-GAN was developed by Chan et al. in December 2020, approximately three months after GRAF [S1].
*   **Paper:** "pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis" [S3].
*   **History:** It was proposed as an improvement over GRAF, addressing its limitations in conditioning and network architecture [S1, S3]. It leverages SIREN (Sinusoidal Representation Networks) [S3].
*   **Usage:** Used for 3D-aware image synthesis on datasets like CelebA, Cats, and CARLA [S3].

## Sources

[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S3] pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis, https://doi.org/10.1109/cvpr46437.2021.00574, fetched 2026-10-26
[S4] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
