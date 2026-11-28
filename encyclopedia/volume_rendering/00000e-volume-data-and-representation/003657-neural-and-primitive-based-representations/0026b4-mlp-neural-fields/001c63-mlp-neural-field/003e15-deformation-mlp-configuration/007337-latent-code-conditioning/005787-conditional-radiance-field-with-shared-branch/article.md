# Conditional Radiance Field with Shared Branch

Also called: CRF with Shared Branch, Modular Conditional Radiance Field

## Overview
The Conditional Radiance Field with Shared Branch is a neural radiance field architecture designed for shape and color disentanglement, enabling efficient instance-specific editing by allowing specific submodules to be fine-tuned independently [S1]. It maps a 3D location $\mathbf{x}$, viewing direction $\mathbf{d}$, and instance-specific latent codes to volumetric density $\sigma$ and radiance $\mathbf{c}$ [S1]. Unlike standard conditional formulations that concatenate all latent codes with positional encodings, this method separates the shape representation into a category-specific shared branch and an instance-specific branch, improving view synthesis quality and scalability to multiple instances [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D position $\mathbf{x}$, viewing direction $\mathbf{d}$, instance-specific shape code $\mathbf{z}^{(s)}$, instance-specific color code $\mathbf{z}^{(c)}$ [S1].
*   **Outputs:** Scalar density $\sigma$, radiance vector $\mathbf{c}$ [S1].

**Data Structures:**
The network consists of five Multi-Layer Perceptrons (MLPs) [S1]. $F_{\text{share}}$, $F_{\text{inst}}$, and $F_{\text{fuse}}$ are 4-layer deep, 256-channel wide MLPs with ReLU activations, outputting 256-dimensional features [S1]. $F_{\text{dens}}$ is a linear layer mapping 256 dimensions to 1 for scalar density [S1]. $F_{\text{rad}}$ is a two-layer MLP [S1]. Latent codes $\mathbf{z}^{(s)}$ and $\mathbf{z}^{(c)}$ are 32-dimensional vectors [S1].

**Numbered Steps:**
1.  **Positional Encoding:** Compute positional encodings $\gamma(\mathbf{x})$ and $\gamma(\mathbf{d})$ for the input position and viewing direction [S1].
2.  **Shared Shape Representation:** Pass $\gamma(\mathbf{x})$ through $F_{\text{share}}$ to obtain the shared shape feature $\mathbf{f}_{\text{share}}$:
    $$ \mathbf{f}_{\text{share}} = F_{\text{share}}(\gamma(\mathbf{x})) $$
3.  **Instance-Specific Shape Representation:** Concatenate $\mathbf{z}^{(s)}$ with $\gamma(\mathbf{x})$ and pass through $F_{\text{inst}}$ to obtain the instance shape feature $\mathbf{f}_{\text{inst}}$:
    $$ \mathbf{f}_{\text{inst}} = F_{\text{inst}}([\mathbf{z}^{(s)}; \gamma(\mathbf{x})]) $$
4.  **Fusion:** Add $\mathbf{f}_{\text{share}}$ and $\mathbf{f}_{\text{inst}}$, then pass the sum through $F_{\text{fuse}}$ to obtain the fused shape feature $\mathbf{f}_{\text{fuse}}$:
    $$ \mathbf{f}_{\text{fuse}} = F_{\text{fuse}}(\mathbf{f}_{\text{share}} + \mathbf{f}_{\text{inst}}) $$
5.  **Density Prediction:** Pass $\mathbf{f}_{\text{fuse}}$ through $F_{\text{dens}}$ to obtain scalar density $\sigma$:
    $$ \sigma = F_{\text{dens}}(\mathbf{f}_{\text{fuse}}) $$
6.  **Radiance Prediction:** Concatenate $\mathbf{f}_{\text{fuse}}$, $\mathbf{z}^{(c)}$, and $\gamma(\mathbf{d})$, then pass through $F_{\text{rad}}$ to obtain radiance $\mathbf{c}$:
    $$ \mathbf{c} = F_{\text{rad}}([\mathbf{f}_{\text{fuse}}; \mathbf{z}^{(c)}; \gamma(\mathbf{d})]) $$

**Rendering:**
The rendering process follows standard volume rendering [Parent](3065-nerf-hyperparameters.md). [G]

**Parameters:**
*   Shape code dimension: 32 [S1].
*   Color code dimension: 32 [S1].
*   MLP depth for $F_{\text{share}}, F_{\text{inst}}, F_{\text{fuse}}$: 4 layers [S1].
*   MLP width: 256 channels [S1].
*   Activation: ReLU [S1].
*   Optimizer: Adam [S1].
*   Training learning rate: $10^{-4}$ [S1].
*   Edit learning rate: $10^{-2}$ [S1].

**Edge Cases and Pitfalls:**
*   **Caching:** During optimization, the set of training rays is small, allowing caching of intermediate representations to accelerate editing [S1].
*   Other edge cases and pitfalls are not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class ConditionalRadianceFieldSharedBranch(nn.Module):
    def __init__(self, enc_dim, latent_dim=32, hidden_dim=256):
        super().__init__()
        # F_share: 4 linear layers, 256 width, ReLU
        self.f_share = nn.Sequential(
            nn.Linear(enc_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim)
        )
        # F_inst: 4 linear layers, 256 width, ReLU
        self.f_inst = nn.Sequential(
            nn.Linear(latent_dim + enc_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim)
        )
        # F_fuse: 4 linear layers, 256 width, ReLU
        self.f_fuse = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim)
        )
        # F_dens: Linear layer
        self.f_dens = nn.Linear(hidden_dim, 1)
        # F_rad: Two-layer MLP (2 linear layers, 256 width, ReLU)
        self.f_rad = nn.Sequential(
            nn.Linear(hidden_dim + latent_dim + enc_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, 3)
        )

    def forward(self, x_enc, d_enc, z_s, z_c):
        # 1. Shared shape representation
        f_share = self.f_share(x_enc)
        # 2. Instance-specific shape representation
        f_inst = self.f_inst(torch.cat([z_s, x_enc], dim=-1))
        # 3. Fuse representations
        f_fuse = self.f_fuse(f_share + f_inst)
        # 4. Predict density
        sigma = self.f_dens(f_fuse)
        # 5. Predict radiance
        c = self.f_rad(torch.cat([f_fuse, z_c, d_enc], dim=-1))
        return sigma, c
```

## Operating characteristics
*   **Time and Memory Complexity:** The shared branch reduces the number of parameters that need to be updated during editing, accelerating color edits by 3.2× and shape edits by 1.9× compared to non-modular approaches [S1]. Caching of intermediate representations is feasible due to the small set of training rays during optimization [S1].
*   **Measured Performance:** On the PhotoShapes dataset, the model with the shared shape branch achieves a PSNR of 35.79, compared to 33.91 for a model without the shared shape branch [S1].
*   **Strengths:** Improves view synthesis quality, enables efficient editing by isolating shape and color updates, and scales better to multiple instances [S1].
*   **Limitations:** Requires careful tuning of latent codes and network architecture to maintain disentanglement [S1].
*   **When to Use:** When efficient instance-specific editing (shape or color) is required, and the dataset contains multiple instances of similar objects [S1].

## Background
The Conditional Radiance Field with Shared Branch was introduced in the paper "Editing Conditional Radiance Fields" (2021) [S1]. The method builds on the original NeRF architecture and aims to provide modular inductive biases for better shape and color disentanglement [S1]. It was developed to address the limitations of single-instance NeRF models and to enable efficient editing of 3D objects represented as radiance fields [S1].

## Sources
[S1] Editing Conditional Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00572, fetched 2026-10-26
