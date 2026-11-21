# GIRAFFE

Also called: Compositional Generative Neural Feature Fields, GIRAFFE Scene Composition

## Overview
GIRAFFE represents scenes as compositions of $N$ entities (objects plus a background), where each entity possesses a separate neural feature field and an affine transformation [S1]. This architecture enables explicit control over the pose, shape, and appearance of individual entities within a generated scene [S1]. It distinguishes itself from sibling methods like [StyleNeRF](8659-stylenerf.md) and [HyperNeRFGAN](8765-hypernerfgan.md) by decoupling scene composition from the generative process, allowing for variable numbers of objects and independent manipulation of entity attributes via latent codes and transformation parameters [S1].

## How it works
GIRAFFE generates images by composing multiple neural feature fields, rendering them into a low-resolution feature map, and upscaling to RGB via a 2D neural renderer [S1]. The method is trained using a GAN objective [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Camera pose $\xi$ (extrinsics) [S1].
    *   Number of entities $N$ (sampled from distribution $p_N$) [S1].
    *   For each entity $i \in \{1, \dots, N\}$:
        *   Shape latent code $\mathbf{z}_s^i \sim \mathcal{N}(0, I)$ [S1].
        *   Appearance latent code $\mathbf{z}_a^i \sim \mathcal{N}(0, I)$ [S1].
        *   Transformation parameters $\mathbf{T}_i = \{\mathbf{s}_i, \mathbf{t}_i, \mathbf{R}_i\}$ (scale, translation, rotation), sampled from $p_T$ [S1].
*   **Outputs:**
    *   Synthesized RGB image $\hat{\mathbf{I}} \in \mathbb{R}^{H \times W \times 3}$ [S1].

### Data Structures
1.  **Feature Field MLP ($h_\theta$):**
    *   **Architecture:** 8 hidden layers, hidden dimension 128, ReLU activations [S1].
    *   **Inputs:**
        *   Positional encoding of canonical position $\gamma(k^{-1}(\mathbf{x}))$ [S1].
        *   Positional encoding of canonical direction $\gamma(k^{-1}(\mathbf{d}))$ [S1].
        *   Shape latent $\mathbf{z}_s$ [S1].
        *   Appearance latent $\mathbf{z}_a$ [S1].
    *   **Outputs:**
        *   Density $\sigma \in \mathbb{R}^+$ [S1].
        *   Feature vector $\mathbf{f} \in \mathbb{R}^{M_f}$ [S1].
    *   **Weight Sharing:** All object feature fields ($i=1 \dots N-1$) share weights $\theta$ [S1]. The background field ($i=N$) uses half the layers and hidden dimension [S1].
    *   **Positional Encoding Dimensions:** $L_x = 2 \cdot 3 \cdot 10$ for position, $L_d = 2 \cdot 3 \cdot 4$ for direction [S1].
2.  **2D Neural Renderer ($\pi_\theta^{\text{neural}}$):**
    *   **Input:** Feature image $\mathbf{I}_V \in \mathbb{R}^{H_V \times W_V \times M_f}$ (resolution $16 \times 16$) [S1].
    *   **Output:** RGB image $\hat{\mathbf{I}} \in \mathbb{R}^{H \times W \times 3}$ (resolution $64 \times 64$ or $256 \times 256$) [S1].
    *   **Architecture:** 2D CNN with leaky ReLU [S1]. Combines nearest neighbor upsampling with $3 \times 3$ convolutions. Maps feature image to RGB at every spatial resolution via $3 \times 3$ conv, adds to previous output via bilinear upsampling (skip connections) [S1]. Final layer applies sigmoid [S1].
3.  **Discriminator ($D_\phi$):**
    *   **Architecture:** CNN with leaky ReLU [S1].

### Parameters
*   $N$: Number of entities. Fixed across dataset or varied [S1].
*   $M_f$: Feature vector dimensionality. 128 for $64 \times 64$ output, 256 for $256 \times 256$ output [S1].
*   $N_s$: Number of sample points per ray. Value: 64 [S1].
*   $\lambda$: R1 gradient penalty weight. Value: 10 [S1].
*   Optimizer: RMSprop [S1].
*   Batch size: 32 [S1].
*   Generator LR: $1 \times 10^{-4}$ (or $2.5 \times 10^{-4}$ for $256 \times 256$) [S1].
*   Discriminator LR: $5 \times 10^{-4}$ [S1].
*   EMA decay: 0.999 for generator weights [S1].

### Steps

1.  **Sample Inputs:** Sample $N \sim p_N$, latent codes $\mathbf{z}_s^i, \mathbf{z}_a^i \sim \mathcal{N}(0, I)$, camera pose $\xi \sim p_\xi$, and transformations $\mathbf{T}_i \sim p_T$ for $i=1 \dots N$ [S1].
2.  **Define Background Transformation:** For the background entity ($i=N$), fix scale $\mathbf{s}_N$ and translation $\mathbf{t}_N$ to span the entire scene and center it at the origin. Other objects have learnable $\mathbf{s}_i, \mathbf{t}_i, \mathbf{R}_i$ [S1].
3.  **Transform Coordinates:** For each entity $i$, define the transformation from object to scene space:
    $$
    k_i(\mathbf{x}) = \mathbf{R}_i \begin{bmatrix} s_{i,1} & 0 & 0 \\ 0 & s_{i,2} & 0 \\ 0 & 0 & s_{i,3} \end{bmatrix} \mathbf{x} + \mathbf{t}_i
    $$
    where $\mathbf{x}$ is a 3D point [S1].
4.  **Evaluate Feature Fields:** For each pixel in the camera view, cast a ray $\mathbf{d}$ and sample $N_s$ points $\mathbf{x}_j$ along the ray. For each sample point $\mathbf{x}_j$ and entity $i$:
    *   Compute canonical coordinates: $\mathbf{x}_{\text{canon}} = k_i^{-1}(\mathbf{x}_j)$, $\mathbf{d}_{\text{canon}} = k_i^{-1}(\mathbf{d})$ [S1].
    *   Apply positional encoding $\gamma(\cdot)$ [S1].
    *   Pass through shared MLP $h_\theta$ with inputs $\gamma(\mathbf{x}_{\text{canon}}), \gamma(\mathbf{d}_{\text{canon}}), \mathbf{z}_s^i, \mathbf{z}_a^i$ to get $(\sigma_{i,j}, \mathbf{f}_{i,j})$ [S1].
5.  **Compose Entities:** Combine the predictions from all $N$ entities using the composition operator $C$:
    $$
    \sigma_j = \sum_{i=1}^N \sigma_{i,j}
    $$
    $$
    \mathbf{f}_j = \frac{1}{\sigma_j} \sum_{i=1}^N \sigma_{i,j} \mathbf{f}_{i,j}
    $$
    [S1].
6.  **Volume Render Feature Map:** Compute the feature vector $\mathbf{f}$ for the pixel by integrating along the ray:
    $$
    \mathbf{f} = \sum_{j=1}^{N_s} \tau_j \alpha_j \mathbf{f}_j
    $$
    where:
    $$
    \alpha_j = 1 - e^{-\sigma_j \delta_j}
    $$
    $$
    \tau_j = \prod_{k=1}^{j-1} (1 - \alpha_k)
    $$
    $$
    \delta_j = \|\mathbf{x}_{j+1} - \mathbf{x}_j\|_2
    $$
    This produces a low-resolution feature image $\mathbf{I}_V$ of size $16 \times 16$ [S1].
7.  **2D Neural Rendering:** Upsample $\mathbf{I}_V$ to the final RGB image $\hat{\mathbf{I}}$ using the 2D CNN $\pi_\theta^{\text{neural}}$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def positional_encoding(x, L):
    """Standard sinusoidal positional encoding: gamma(x) = (sin(2^0 pi x), cos(2^0 pi x), ..., sin(2^{L-1} pi x), cos(2^{L-1} pi x))"""
    freq_bands = torch.linspace(0, L-1, L, device=x.device)
    scaled = freq_bands.view(1, 1, -1) * torch.pi * x.unsqueeze(-1)
    return torch.cat([torch.sin(scaled), torch.cos(scaled)], dim=-1)

class FeatureFieldMLP(nn.Module):
    def __init__(self, hidden_dim, M_f, num_layers=8, is_background=False):
        super().__init__()
        if is_background:
            num_layers = num_layers // 2
            hidden_dim = hidden_dim // 2
        self.layers = nn.ModuleList([nn.Linear(hidden_dim, hidden_dim) for _ in range(num_layers)])
        self.relu = nn.ReLU()
        self.density_head = nn.Linear(hidden_dim, 1)
        self.feature_head = nn.Linear(hidden_dim, M_f)

    def forward(self, x_enc, d_enc, z_s, z_a):
        h = torch.cat([x_enc, d_enc, z_s, z_a], dim=-1)
        for layer in self.layers:
            h = self.relu(layer(h))
        sigma = self.density_head(h)
        f = self.feature_head(h)
        return sigma, f

class NeuralRenderer(nn.Module):
    def __init__(self, M_f, target_res):
        super().__init__()
        self.target_res = target_res
        # 2D CNN with leaky ReLU, nearest neighbor upsampling, 3x3 convs, skip connections
        self.conv1 = nn.Conv2d(M_f, 64, 3, padding=1)
        self.conv2 = nn.Conv2d(64, 64, 3, padding=1)
        self.conv3 = nn.Conv2d(64, 128, 3, padding=1)
        self.conv4 = nn.Conv2d(128, 128, 3, padding=1)
        self.conv5 = nn.Conv2d(128, 3, 3, padding=1)
        self.leaky_relu = nn.LeakyReLU(0.2)

    def forward(self, I_V):
        # I_V: (B, M_f, 16, 16)
        x = self.leaky_relu(self.conv1(I_V))
        x = F.interpolate(x, scale_factor=2, mode='nearest')
        x = self.leaky_relu(self.conv2(x))
        x = F.interpolate(x, scale_factor=2, mode='nearest')
        # Skip connections via bilinear upsampling as per source
        x = x + F.interpolate(self.leaky_relu(self.conv3(I_V)), scale_factor=4, mode='bilinear', align_corners=False)
        x = self.leaky_relu(self.conv4(x))
        out = self.conv5(x)
        return torch.sigmoid(out)

class GIRAFFEGenerator(nn.Module):
    def __init__(self, M_f, L_x, L_d, max_N=5):
        super().__init__()
        self.M_f = M_f
        self.max_N = max_N
        self.L_x = L_x
        self.L_d = L_d
        self.feature_field = FeatureFieldMLP(hidden_dim=128, M_f=M_f, num_layers=8, is_background=False)
        self.bg_field = FeatureFieldMLP(hidden_dim=128, M_f=M_f, num_layers=8, is_background=True)
        self.renderer = NeuralRenderer(M_f, target_res=64)

    def inverse_transform(self, x_scene, s, t, R):
        # k_inv(x) = S^{-1} R^T (x - t)
        x_centered = x_scene - t.unsqueeze(0)
        x_rotated = torch.matmul(x_centered, R)
        return x_rotated / s.unsqueeze(0)

    def forward(self, ray_origins, ray_directions, N, z_s, z_a, T_list):
        # ray_origins: (B, H, W, 3), ray_directions: (B, H, W, 3)
        # N: (B,), z_s: (B, max_N, latent), z_a: (B, max_N, latent)
        # T_list: list of dicts with s, t, R for each entity

        all_sigmas = []
        all_features = []

        for i in range(self.max_N):
            if i >= N.max():
                break
            s = T_list[i]['s']
            t = T_list[i]['t']
            R = T_list[i]['R']
            is_bg = (i == N - 1)
            zs = z_s[:, i, :]
            za = z_a[:, i, :]

            mlp = self.bg_field if is_bg else self.feature_field

            x_canon = self.inverse_transform(ray_origins, s, t, R)
            d_canon = self.inverse_transform(ray_directions, s, t, R)

            x_enc = positional_encoding(x_canon, self.L_x)
            d_enc = positional_encoding(d_canon, self.L_d)

            sigma, f = mlp(x_enc, d_enc, zs, za)
            all_sigmas.append(sigma)
            all_features.append(f)

        # Compose
        sigma_total = sum(all_sigmas)
        feature_sum = sum(sigma_i * f_i for sigma_i, f_i in zip(all_sigmas, all_features))
        f_composed = feature_sum / (sigma_total + 1e-8) # Epsilon guard [G]

        # Volume Render
        delta = torch.norm(ray_origins[:, 1:] - ray_origins[:, :-1], dim=-1, keepdim=True)
        delta = torch.cat([delta, delta[:, -1:]], dim=1) # Pad last delta
        alpha = 1 - torch.exp(-sigma_total * delta)
        tau = torch.cat([torch.ones_like(alpha[:, :1]), torch.cumprod(1 - alpha[:, :-1], dim=1)], dim=1)
        f_rendered = torch.sum(tau * alpha * f_composed, dim=1, keepdim=True) # (B, 1, 1, M_f)

        # Reshape for 2D renderer (assume input grid is 16x16)
        I_V = f_rendered.permute(0, 3, 1, 2)
        I_RGB = self.renderer(I_V)
        return I_RGB
```

### Edge Cases and Pitfalls
*   **Zero Density:** If $\sigma_j = 0$ for all entities at a sample point, the composition operator divides by zero. A small epsilon is added to the denominator [G].
*   **Background Transformation:** The background entity's scale and translation are fixed to span the scene [S1]. Incorrect initialization or updates to these parameters can cause the background to disappear or dominate the scene [S1].
*   **Resolution Mismatch:** The feature image is rendered at $16 \times 16$ and upsampled [S1]. If the neural renderer is not properly trained, artifacts may appear during upsampling [S1].
*   **Variable N:** The model must handle variable numbers of entities $N$ [S1]. The pseudocode assumes a fixed maximum $N$ and masks out unused entities [S1].

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to $N \cdot N_s \cdot 16 \cdot 16$ for the volume rendering step, plus the cost of the 2D neural renderer for upsampling to the final resolution [S1].
*   **Memory Complexity:** Memory usage is dominated by the MLP weights and the feature map. The feature map is small ($16 \times 16 \times M_f$), reducing memory compared to full-resolution RGB rendering [S1].
*   **Performance:** GIRAFFE achieves high-quality image synthesis on datasets like Chairs, Cats, and CelebA [S1]. It allows for disentangled control of object pose, shape, and appearance [S1].
*   **Strengths:** Explicit scene composition, control over individual entities, efficient rendering via low-resolution feature maps [S1].
*   **Limitations:** Assumes a limited number of entities. Background handling is simplified [S1].

## Background
GIRAFFE was introduced in the paper "GIRAFFE: Representing Scenes as Compositional Generative Neural Feature Fields" [S1]. It extends previous NeRF-based generative models by introducing a compositional structure that allows for the generation of scenes with multiple objects, each with independent attributes [S1].

## Sources
[S1] GIRAFFE: Representing Scenes as Compositional Generative Neural Feature Fields, https://github.com/chaerinkong/paper-reviews/issues/63, fetched 2026-10-28
