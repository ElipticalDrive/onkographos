# pi-GAN

Also called: Periodic Implicit Generative Adversarial Networks, π-GAN

## Overview
pi-GAN is a generative adversarial network (GAN) approach for unsupervised 3D-aware image synthesis that learns a view-consistent 5D spatio-angular radiance field from unlabeled 2D images [S1]. It differs from standard 2D GANs by conditioning a SIREN-based implicit radiance field on a latent noise vector to produce volumetric density and color, which are then rendered via volume rendering [S1]. Unlike GRAF, which concatenates noise codes, pi-GAN uses a StyleGAN-inspired mapping network with Feature-wise Linear Modulation (FiLM) to condition the SIREN, and it employs a progressive growing strategy to stabilize training and accelerate convergence [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A latent noise vector $z \sim p_z$ (typically sampled from a standard normal distribution) and a camera pose $\xi$ (defined by origin $o$ and viewing direction $d$).
*   **Output:** A synthesized 2D image $I$ rendered from the camera pose $\xi$.

**Data Structures:**
1.  **Mapping Network:** A simple Multi-Layer Perceptron (MLP) with ReLU activations.
    *   Input: Latent vector $z$.
    *   Output: A set of frequency parameters $\{\gamma_i\}$ and phase shift parameters $\{\beta_i\}$ for each layer $i$ of the SIREN backbone.
2.  **SIREN Backbone ($\Phi$):** A neural network using periodic sine activations.
    *   Input: 3D spatial coordinate $\mathbf{x} = (x, y, z)$.
    *   Output: Intermediate feature representation $\Phi(\mathbf{x})$.
3.  **Density Head:** A linear layer.
    *   Input: $\Phi(\mathbf{x})$.
    *   Output: Scalar density $\sigma(\mathbf{x}) \in \mathbb{R}$.
4.  **Color Head:** A linear layer with an activation function $\phi_c$ (typically ReLU or similar, though the source denotes it generally as $\phi_c$).
    *   Input: Concatenation of $\Phi(\mathbf{x})$ and viewing direction $d$.
    *   Output: View-dependent color $\mathbf{c}(\mathbf{x}, \mathbf{d}) \in \mathbb{R}^3$.
5.  **Discriminator:** A traditional convolutional neural network (CNN) that takes 2D images as input and outputs a scalar probability of being real.

**Numbered Steps:**

1.  **Latent Mapping:** The mapping network processes the input noise vector $z$ to generate layer-specific conditioning parameters. For each layer $i$ of the SIREN, it outputs a frequency $\gamma_i$ and a phase shift $\beta_i$ [S1].
2.  **SIREN Evaluation:** The SIREN backbone computes the implicit representation $\Phi(\mathbf{x})$ for a given 3D point $\mathbf{x}$. The $i$-th layer transformation is defined as:
    $$ \phi_i(\mathbf{x}_i) = \sin(\gamma_i \cdot (\mathbf{W}_i \mathbf{x}_i + \mathbf{b}_i) + \beta_i) $$
    where $\mathbf{W}_i$ and $\mathbf{b}_i$ are the learned weights and biases of the SIREN layer, and $\mathbf{x}_i$ is the input to layer $i$ [S1]. The full backbone is the composition $\Phi(\mathbf{x}) = \phi_{n-1} \circ \dots \circ \phi_0(\mathbf{x})$ [S1].
3.  **Radiance Field Prediction:**
    *   Density is predicted as a linear function of the backbone output:
        $$ \sigma(\mathbf{x}) = \mathbf{W}_\sigma \Phi(\mathbf{x}) + \mathbf{b}_\sigma $$
        where $\mathbf{W}_\sigma$ and $\mathbf{b}_\sigma$ are learned parameters [S1].
    *   Color is predicted by concatenating the backbone output with the viewing direction $d$:
        $$ \mathbf{c}(\mathbf{x}, \mathbf{d}) = \mathbf{W}_c \phi_c \left( [\Phi(\mathbf{x}), \mathbf{d}]^T \right) + \mathbf{b}_c $$
        where $\mathbf{W}_c$ and $\mathbf{b}_c$ are learned parameters, and $\phi_c$ is the activation function of the color head [S1].
4.  **Volume Rendering:** Rays are cast from the camera origin $o$ through each pixel. For a ray $\mathbf{r}(t) = o + t\mathbf{d}$, the pixel color $C$ is computed by integrating density and color along the ray segment between near bound $t_n$ and far bound $t_f$. The standard volume rendering equation is used:
    $$ C(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(\mathbf{r}(t)) \mathbf{c}(\mathbf{r}(t), \mathbf{d}) \, dt $$
    where $T(t) = \exp\left(-\int_{t_n}^t \sigma(\mathbf{r}(s)) \, ds\right)$ is the accumulated transmittance [S1]. In practice, this integral is approximated by sampling points along the ray and summing the contribution of each sample weighted by its transmittance and opacity [S1].
5.  **Adversarial Training:** The rendered image is passed to a convolutional discriminator. The generator (SIREN + Mapping Network) is trained to minimize the adversarial loss (fooling the discriminator), while the discriminator is trained to maximize the classification accuracy between real and generated images [S1].
6.  **Progressive Growing:** Training employs a progressive growing strategy, gradually increasing the resolution of the rendered images and the complexity of the discriminator to accelerate training and stabilize the GAN dynamics [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MappingNetwork(nn.Module):
    """
    Simple ReLU MLP that maps latent vector z to SIREN parameters.
    """
    def __init__(self, latent_dim, num_layers, hidden_dim):
        super().__init__()
        self.num_layers = num_layers
        # Outputs 2 * num_layers parameters: gamma and beta for each layer
        self.fc = nn.Sequential(
            nn.Linear(latent_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, num_layers * 2)
        )

    def forward(self, z):
        params = self.fc(z)
        # Split into gammas and betas
        gammas = params[:, :self.num_layers]
        betas = params[:, self.num_layers:]
        return gammas, betas

class SIRENLayer(nn.Module):
    def __init__(self, in_features, out_features, gamma, beta):
        super().__init__()
        self.linear = nn.Linear(in_features, out_features)
        self.gamma = gamma
        self.beta = beta
        # Initialize weights for sine activation
        self._init_weights()

    def _init_weights(self):
        # Standard SIREN initialization: uniform distribution scaled by 1/sqrt(in_features)
        # and divided by gamma
        with torch.no_grad():
            self.linear.weight.uniform_(
                -1 / self.linear.in_features / self.gamma,
                1 / self.linear.in_features / self.gamma
            )
            self.linear.bias.uniform_(
                -1 / self.linear.in_features / self.gamma,
                1 / self.linear.in_features / self.gamma
            )

    def forward(self, x):
        return torch.sin(self.gamma * self.linear(x) + self.beta)

class SIRENBackbone(nn.Module):
    def __init__(self, in_dim, out_dim, num_layers, hidden_dim):
        super().__init__()
        self.num_layers = num_layers
        self.hidden_dim = hidden_dim
        self.layers = nn.ModuleList()
        
        # First layer
        self.layers.append(SIRENLayer(in_dim, hidden_dim, gamma=1.0, beta=0.0))
        # Hidden layers
        for _ in range(num_layers - 2):
            self.layers.append(SIRENLayer(hidden_dim, hidden_dim, gamma=1.0, beta=0.0))
        # Last layer
        self.layers.append(SIRENLayer(hidden_dim, out_dim, gamma=1.0, beta=0.0))

    def forward(self, x, gammas, betas):
        # x: [B, 3]
        # gammas, betas: [B, num_layers]
        h = x
        for i, layer in enumerate(self.layers):
            # Update layer parameters with FiLM conditioning
            layer.gamma = gammas[:, i]
            layer.beta = betas[:, i]
            h = layer(h)
        return h

class PiGANGenerator(nn.Module):
    def __init__(self, latent_dim, num_siren_layers, siren_hidden_dim):
        super().__init__()
        self.mapping = MappingNetwork(latent_dim, num_siren_layers, siren_hidden_dim)
        # SIREN outputs a feature vector. We split this into density and color features.
        # Let's assume SIREN outputs a feature vector of size F.
        # Density head: F -> 1
        # Color head: F + 3 (direction) -> 3
        self.siren_out_dim = siren_hidden_dim 
        self.siren = SIRENBackbone(3, self.siren_out_dim, num_siren_layers, siren_hidden_dim)
        
        self.density_head = nn.Linear(self.siren_out_dim, 1)
        self.color_head = nn.Sequential(
            nn.Linear(self.siren_out_dim + 3, siren_hidden_dim),
            nn.ReLU(),
            nn.Linear(siren_hidden_dim, 3)
        )

    def forward(self, z, rays_o, rays_d):
        """
        z: [B, latent_dim]
        rays_o: [B, H, W, 3] camera origins
        rays_d: [B, H, W, 3] camera directions (normalized)
        """
        B, H, W = rays_o.shape[:3]
        
        # 1. Map latent to SIREN parameters
        gammas, betas = self.mapping(z) # [B, num_layers]
        
        # 2. Flatten spatial dimensions for SIREN evaluation
        # We need to sample points along rays. For simplicity, assume single sample per ray or fixed steps.
        # Here we demonstrate the forward pass for a set of 3D points X.
        # In a full implementation, X would be generated by ray marching.
        # For this pseudocode, we assume X is provided as [B, N, 3]
        pass

    def render_ray(self, z, ray_o, ray_d, t_near, t_far, num_samples):
        """
        Renders a single ray.
        ray_o: [3]
        ray_d: [3]
        """
        # Sample points along the ray
        t_vals = torch.linspace(t_near, t_far, num_samples)
        xyz = ray_o.unsqueeze(0) + t_vals.unsqueeze(1) * ray_d.unsqueeze(0) # [1, N, 3]
        
        # Get SIREN parameters for this specific z (assuming z is [1, latent_dim])
        gammas, betas = self.mapping(z)
        
        # Evaluate SIREN
        phi = self.siren(xyz, gammas, betas) # [1, N, F]
        
        # Predict density
        sigma = torch.sigmoid(self.density_head(phi)) # [1, N, 1] (ensure positive)
        
        # Predict color
        # Concatenate phi and viewing direction
        d_expanded = ray_d.unsqueeze(0).unsqueeze(0).expand_as(phi[:, :, :3]) # [1, N, 3]
        color_input = torch.cat([phi, d_expanded], dim=-1) # [1, N, F+3]
        c = self.color_head(color_input) # [1, N, 3]
        
        # Volume Rendering
        # Compute deltas
        deltas = t_vals[1:] - t_vals[:-1]
        delta_inf = 1e10 * torch.ones_like(deltas[-1:])
        deltas = torch.cat([deltas, delta_inf], dim=0) # [N]
        
        # Alpha
        alpha = 1 - torch.exp(-sigma.squeeze() * deltas) # [N]
        
        # Transmittance
        T = torch.cumprod(torch.cat([torch.ones_like(alpha[:1]), 1 - alpha + 1e-10], dim=0), dim=0)[:-1] # [N]
        
        # Weight
        weights = alpha * T # [N]
        
        # Color
        C = torch.sum(weights.unsqueeze(-1) * c.squeeze(), dim=0) # [3]
        
        return C

class Discriminator(nn.Module):
    def __init__(self, img_size=256, channels=3):
        super().__init__()
        # Standard CNN discriminator
        self.net = nn.Sequential(
            nn.Conv2d(channels, 64, 4, 2, 1),
            nn.LeakyReLU(0.2),
            nn.Conv2d(64, 128, 4, 2, 1),
            nn.LeakyReLU(0.2),
            # ... more layers ...
            nn.AdaptiveAvgPool2d(1),
            nn.Flatten(),
            nn.Linear(128, 1)
        )

    def forward(self, x):
        return self.net(x)
```

**Parameters:**
*   $\gamma_i$: Frequency parameter for the sine activation in SIREN layer $i$. Output by the mapping network. Typical values are learned, but SIREN initialization often uses uniform distributions scaled by $1/\sqrt{M_i}$ [S1].
*   $\beta_i$: Phase shift parameter for the sine activation in SIREN layer $i$. Output by the mapping network. Initialized to 0 [S1].
*   $z$: Latent noise vector. Dimension depends on the mapping network input [S1].
*   $\xi$: Camera pose. Defined by origin $o$ and direction $d$ [S1].

**Edge Cases and Pitfalls:**
*   **Local Overfitting:** SIREN networks are prone to local overfitting to the last batch if sufficient coverage of the 3D space is not maintained during training. This is a key reason why pi-GAN does not use a patch-based discriminator (unlike GRAF) [S1].
*   **Ray Marching Resolution:** The quality of the rendered image depends on the number of samples along the ray. Insufficient sampling can lead to aliasing or inaccurate density integration [S1].
*   **Density Clamping:** Density values $\sigma(\mathbf{x})$ must be non-negative. The density head typically uses a sigmoid or softplus activation to ensure this [S1].

## Operating characteristics
*   **Time Complexity:** Training is computationally expensive due to the need to evaluate the SIREN for every sample point along every ray for every image in the batch. The progressive growing strategy helps mitigate this by starting with low resolutions [S1].
*   **Memory Complexity:** Memory usage is dominated by the SIREN parameters and the intermediate activations during volume rendering. The mapping network is lightweight [S1].
*   **Performance:** pi-GAN achieves state-of-the-art results for 3D-aware image synthesis on datasets such as CelebA, Cats, and CARLA [S1].
*   **Strengths:**
    *   Generates high-quality, view-consistent images.
    *   Unsupervised learning from 2D images.
    *   Allows extraction of interpretable 3D proxy shapes via marching cubes [S1].
*   **Limitations:**
    *   Proxy shapes extracted via marching cubes may not be as high quality as those from supervised single-view reconstruction methods [S1].
    *   Training can be unstable without careful hyperparameter tuning and progressive growing [S1].
*   **When to Use:** When unsupervised 3D-aware image synthesis is required from 2D data, and view consistency is critical.
*   **When Not to Use:** When high-fidelity explicit 3D geometry is required without post-processing, or when supervised data is available for shape reconstruction.

## Background
pi-GAN was introduced in 2021 by Tero Karras et al. in the paper "pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis" [S1]. It builds upon the success of StyleGAN and the neural implicit representation techniques like SIREN. The method addresses the limitations of previous 3D-aware GANs, such as view-inconsistent rendering and lack of expressive power, by combining periodic activation functions with a FiLM-based conditioning mechanism [S1].

## Sources
[S1] pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis, https://doi.org/10.1109/cvpr46437.2021.00574, fetched 2026-10-26
