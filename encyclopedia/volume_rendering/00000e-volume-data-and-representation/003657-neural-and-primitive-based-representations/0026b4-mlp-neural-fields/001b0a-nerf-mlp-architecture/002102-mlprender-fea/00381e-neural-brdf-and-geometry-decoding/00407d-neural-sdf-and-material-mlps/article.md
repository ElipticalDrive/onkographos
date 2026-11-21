# Neural SDF and Material MLPs

Also called: IRON Pipeline Neural Architectures, IRON SDF and Material Networks, Neural Field Architecture & Training Scheme, Neural SDF

## Overview
This method defines the specific multi-layer perceptron (MLP) architectures and training schedules used to decode signed distance functions (SDF) and spatially-varying material properties (diffuse albedo, specular albedo, roughness) in neural implicit surface reconstruction pipelines, most notably the IRON framework. It solves the problem of spectral bias and training instability in single-network approaches by separating geometry and material decoding into distinct networks, allowing the material decoder to access high-level geometric context via latent feature vectors [S1, S3]. It is distinguished from sibling methods like [TensoRF Feature Decoding Architecture](8181-tensorf-feature-decoding-architecture.md) by its explicit use of positional encoding frequencies tailored to specific material parameters and its two-stage optimization strategy [S1, S3].

## How it works
**Inputs and Outputs:**
The method takes 3D coordinates $x \in \mathbb{R}^3$ as the primary input. Depending on the specific MLP, it may also take surface normals $n$, view directions $v$, or latent geometric features $f_{geo}$ [S1, S3].
Outputs include:
- SDF value $s$ and geometric feature vector $f_{geo}$ [S1, S2, S3].
- Diffuse albedo $\rho_d$ [S1, S2].
- Specular albedo $\rho_s$ (or $\kappa$) [S1, S2].
- Roughness $\rho$ (or $\alpha$) [S1, S2, S4].
- Emitter semantic values [S3].

**Data Structures:**
The core data structure is a Multi-Layer Perceptron (MLP).
- **Neural SDF MLP ($F_d$ or $f_{\Theta_g}$):** 8 layers, hidden dimension 256. Skip connection from input to the 4th layer [S1, S2, S3].
- **Neural Diffuse Albedo MLP ($F_{\rho}$ or $f_{\Theta_d}$):** 8 layers, hidden dimension 256. Skip connection from input to the 4th layer [S1, S2].
- **Neural Specular Albedo MLP ($F_{a}$ or $f_{\Theta_s}$):** 4 layers, hidden dimension 256 [S1, S2].
- **Neural Roughness MLP ($F_{\rho}$ or $f_{\Theta_r}$):** 4 layers, hidden dimension 256 [S1, S2].
- **Radiance MLP ($F_c$):** 4 layers, hidden dimension 256 [S3].
- **Emitter Semantic MLP ($F_e$):** 2 layers, hidden dimension 128 [S3].
- **Material Fields ($F_{\rho}, F_a$ in [S3]):** 3 layers, hidden dimension 64 [S3].

**Positional Encoding:**
Sinusoidal positional encoding is applied to inputs to mitigate spectral bias [S1, S3].
- Input $x$: 6 frequencies for SDF, Specular, Roughness [S1, S2, S3]. 10 frequencies for Diffuse Albedo [S1, S2].
- View direction $v$: 4 frequencies [S3].
- Surface normal $n$: 4 frequencies (for Diffuse Albedo) [S1].

**Numbered Steps:**
1. **Geometry Encoding:** Encode input coordinate $x$ with 6-frequency sinusoidal positional encoding. Pass through the 8-layer SDF MLP ($F_d$) with skip connections at layer 4. Output the scalar SDF value $s$ and the 256-dimensional latent geometric feature vector $f_{geo}$ [S1, S2, S3].
2. **Material Decoding (Diffuse):** Encode $x$ with 10 frequencies and $n$ with 4 frequencies. Concatenate with $f_{geo}$. Pass through the 8-layer Diffuse Albedo MLP. Output diffuse albedo $\rho_d$ [S1, S2].
3. **Material Decoding (Specular/Roughness):** Encode $x$ with 6 frequencies. Concatenate with $f_{geo}$ (and optionally $n$). Pass through the respective 4-layer MLPs. Output specular albedo $\rho_s$ and roughness $\rho$ [S1, S2].
4. **Training Schedule:**
   - **Stage 1:** Jointly optimize SDF network $F_d$, radiance network $F_c$, and emitter semantic network $F_e$ for 200k iterations [S3].
   - **Stage 2:** Jointly optimize material fields ($F_{\rho}, F_a$) and light fields for 100k iterations [S3].

**Formulas:**
- **Positional Encoding:** $\gamma(x) = (\sin(2^0 \pi x), \cos(2^0 \pi x), \dots, \sin(2^{L-1} \pi x), \cos(2^{L-1} \pi x))$ [S3, implied by [18]].
- **vMF BRDF Approximation (for Material Parameterization):**
  $$f(l, -d, x_s) \approx G(l, l_r, x_s) = \alpha \exp(\rho(x_s)(l \cdot l_r - 1))$$
  Where $l$ is the light direction, $d$ is the view direction, $l_r$ is the reflection direction, $\alpha$ is specular tint, and $\rho$ is roughness [S4].
- **Reflection Direction:**
  $$l_r = 2(-d \cdot n)n + d$$
  Where $n$ is the surface normal [S4].

**Parameters:**
- SDF MLP width: 256 [S1, S2, S3].
- Diffuse MLP width: 256 [S1, S2].
- Specular/Roughness MLP width: 256 [S1, S2].
- Radiance MLP width: 256 [S3].
- Emitter MLP width: 128 [S3].
- Material fields width: 64 [S3].
- Sample rate $N$: 16 for Monte Carlo integration [S3].
- Training iterations: 200k (Stage 1), 100k (Stage 2) [S3].

**Edge Cases and Pitfalls:**
- **Spectral Bias:** Without positional encoding, MLPs struggle to represent high-frequency details. The specific frequency counts (6 for geometry, 10 for diffuse) are critical [S1, S3].
- **Initialization:** Geometric initialization is performed before optimization to stabilize training [S2].
- **Loss Weights:** In Stage 1, loss weights ($\lambda_1, \lambda_4, \lambda_5$) for eikonal, smooth, and bubble losses are adjusted during warm-up, bubble, and smooth steps [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class SinusoidalPositionalEncoding(nn.Module):
    def __init__(self, num_frequencies):
        super().__init__()
        self.num_frequencies = num_frequencies

    def forward(self, x):
        # x shape: (..., 3)
        # Output shape: (..., 3 * num_frequencies * 2)
        device = x.device
        frequencies = torch.arange(0, self.num_frequencies, device=device).float()
        # 2^i * pi * x
        angular_speeds = (2.0 ** frequencies) * math.pi
        angular_speeds = angular_speeds.view(1, 1, -1) # (1, 1, num_frequencies)
        
        # Expand x to (..., 1, 3) and angular_speeds to (1, num_frequencies, 1)
        # Actually, standard implementation:
        # x_embedded = x.unsqueeze(-1) * angular_speeds.unsqueeze(-2) # (..., 3, num_frequencies)
        # This logic is simplified for clarity; standard NeRF PE is:
        # freq_bands = 2**torch.arange(num_frequencies) * math.pi
        # x_embedded = x.unsqueeze(-1) * freq_bands.unsqueeze(0) # (..., 3, num_frequencies)
        # out = torch.cat([torch.sin(x_embedded), torch.cos(x_embedded)], dim=-1)
        
        # Correct standard NeRF PE implementation:
        freq_bands = 2.0 ** torch.arange(self.num_frequencies, device=device).float() * math.pi
        # freq_bands: (num_frequencies,)
        # x: (..., 3)
        # x.unsqueeze(-1): (..., 3, 1)
        # freq_bands.unsqueeze(0).unsqueeze(0): (1, 1, num_frequencies)
        x_embedded = x.unsqueeze(-1) * freq_bands.unsqueeze(0).unsqueeze(0) # (..., 3, num_frequencies)
        x_embedded = x_embedded.view(*x.shape[:-1], -1) # (..., 3 * num_frequencies)
        return torch.cat([torch.sin(x_embedded), torch.cos(x_embedded)], dim=-1)

class SkipConnectedMLP(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim, num_layers, skip_layer_idx):
        super().__init__()
        self.num_layers = num_layers
        self.skip_layer_idx = skip_layer_idx
        self.layers = nn.ModuleList()
        
        # First layer
        self.layers.append(nn.Linear(input_dim, hidden_dim))
        # Hidden layers
        for _ in range(num_layers - 2):
            self.layers.append(nn.Linear(hidden_dim, hidden_dim))
        # Output layer
        self.layers.append(nn.Linear(hidden_dim, output_dim))
        
        self.relu = nn.ReLU()

    def forward(self, x):
        h = x
        for i, layer in enumerate(self.layers):
            if i == self.skip_layer_idx:
                h = torch.cat([h, x], dim=-1)
            h = layer(h)
            if i < self.num_layers - 1:
                h = self.relu(h)
        return h

def create_sdf_network():
    # 8 layers, width 256, skip at 4th layer (index 3 if 0-indexed layers? 
    # Note: "skip connection at the 4th layer" usually means input is added to the 4th hidden layer.
    # If layers are 0..7, and skip is at 4th layer, it likely means after 3 linear layers?
    # Standard VolSDF/NeRF: Skip connection from input to every hidden layer or specific ones.
    # S1 says "skip connection at the 4th layer". S3 says "skip connection is used in the 4th layer in Fd".
    # Assuming 0-indexed: Layer 0, 1, 2, 3(skip), 4, 5, 6, 7.
    # Let's assume skip is added to the input of the 4th linear layer (index 3).
    return SkipConnectedMLP(input_dim=18, hidden_dim=256, output_dim=257, num_layers=8, skip_layer_idx=3)

def create_diffuse_network():
    # 8 layers, width 256, skip at 4th layer.
    # Input: x (10 freq -> 60) + n (4 freq -> 24) + f_geo (256) = 340
    return SkipConnectedMLP(input_dim=340, hidden_dim=256, output_dim=1, num_layers=8, skip_layer_idx=3)

def create_specular_network():
    # 4 layers, width 256.
    # Input: x (6 freq -> 36) + f_geo (256) = 292
    return SkipConnectedMLP(input_dim=292, hidden_dim=256, output_dim=1, num_layers=4, skip_layer_idx=None)

def create_roughness_network():
    # 4 layers, width 256.
    return SkipConnectedMLP(input_dim=292, hidden_dim=256, output_dim=1, num_layers=4, skip_layer_idx=None)

def compute_vmf_brdf(light_dir, view_dir, normal, roughness, specular_tint):
    # light_dir, view_dir, normal: (..., 3)
    # roughness, specular_tint: (..., 1)
    
    # Compute reflection direction lr = 2(-d . n)n + d
    # d is view_dir
    dot_dn = torch.sum(view_dir * normal, dim=-1, keepdim=True)
    lr = 2 * (-dot_dn) * normal + view_dir
    
    # Normalize lr (should be unit if d and n are unit)
    lr = torch.nn.functional.normalize(lr, dim=-1)
    
    # l . lr
    l_dot_lr = torch.sum(light_dir * lr, dim=-1, keepdim=True)
    
    # G(l, lr, xs) = alpha * exp( rho * (l . lr - 1) )
    # Note: S4 uses alpha for specular tint and rho for roughness.
    # Formula: alpha * exp( rho * (l . lr - 1) )
    brdf = specular_tint * torch.exp(roughness * (l_dot_lr - 1))
    
    return brdf
```

## Subtopics

### [Geometry and material MLP architecture](14140-geometry-and-material-mlp-architecture.md)
This architecture defines the neural network components for decoding signed distance functions (SDF) and material properties (diffuse albedo, specular coefficients) in neural implicit surface reconstruction. It separates geometry and material decoding into two distinct multi-layer perceptrons (MLPs) to improve training stability and rendering fidelity. It is distinguished from single-MLP approaches by its explicit feature vector passing from the geometry network to the material network, allowing the material decoder to access high-level geometric context.

### [Neural diffuse albedo](15344-neural-diffuse-albedo.md)
Neural diffuse albedo is a method for predicting the spatially-varying diffuse reflectance ($\rho_d$) of a surface point using a multi-layer perceptron (MLP). It solves the problem of recovering physically-based material properties from inverse rendering pipelines by mapping 3D geometry and local features to albedo values. It is distinguished from sibling material decoders by its specific input composition, which includes surface normals and geometric features, and its higher-frequency positional encoding for coordinates compared to specular or roughness predictors.

### Material Parameterization with vMF BRDF
This variant parameterizes the spatially-varying material of a surface point $x_s$ as roughness $\rho \in [0, 1]$, diffuse color $C_d \in [0, 1]$, and specular tint $\alpha \in [0, 1]$, output by a material MLP network $M$ using softplus activation [S4]. It approximates the BRDF as a von Mises-Fisher (vMF) distribution: $f(l, -d, x_s) \approx G(l, l_r, x_s) = \alpha \exp(\rho(x_s)(l \cdot l_r - 1))$, where $l_r$ is the reflection direction computed as $2(-d \cdot n)n + d$ [S4]. This allows the model to behave as an interpolation function for incoming illumination inside the roughness-varying specular lobe [S4].

## Operating characteristics
- **Time Complexity:** Training is computationally intensive. Stage 1 (geometry/radiance) takes ~15 hours per scene on a single NVIDIA Tesla V100 GPU for 200k iterations [S3]. Stage 2 takes additional time for 100k iterations [S3].
- **Memory Complexity:** Depends on the number of parameters in the MLPs. With hidden dimensions of 256 and 8 layers, the SDF MLP has approximately $256 \times (18+256) + 256^2 + \dots$ parameters. The latent code $z(x)$ has dimension 256 [S3].
- **Measured Performance:** Rendering on an NVIDIA RTX 3090 GPU with 4096 samples per pixel (spp) takes ~15 seconds per image [S3].
- **Scaling:** The computational cost of predicting incident radiance grows proportional to the sample rate $N$ [S3]. A trade-off between quality and performance is achieved with $N=16$ [S3].
- **Strengths:** Separating geometry and material improves training stability and allows for detailed material recovery [S1]. The use of positional encoding mitigates spectral bias [S1, S3].
- **Limitations:** High computational cost for training and rendering [S3]. Requires careful tuning of loss weights and training stages [S3].
- **When to use:** When high-fidelity material and geometry reconstruction from photometric images is required, and computational resources are available [S1, S3].
- **When not to use:** For real-time applications or resource-constrained environments due to the high training and rendering costs [S3].

## Background
The method originates from the IRON (Inverse Rendering by Optimizing Neural SDFs and Materials) pipeline and related works like Neus and I2-SDF [S1, S3]. It builds on the VolSDF architecture for neural SDFs [S3]. The approach was developed to address the challenges of inverse rendering, specifically the joint optimization of geometry, lighting, and material properties from 2D images [S1, S3]. It is used in computer vision and graphics for tasks such as 3D reconstruction, intrinsic image decomposition, and realistic rendering [S1, S3].

## Sources
[S1] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
[S2] PIR: Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization, https://doi.org/10.1109/3dv66043.2025.00055, fetched 2026-10-26
[S3] I 2 -SDF: Intrinsic Indoor Scene Reconstruction and Editing via Raytracing in Neural SDFs, https://doi.org/10.1109/cvpr52729.2023.01202, fetched 2026-11-01
[S4] A Pre-convolved Representation for Plug-and-Play Neural Illumination Fields, https://doi.org/10.1609/aaai.v38i7.28618, fetched 2026-10-29
