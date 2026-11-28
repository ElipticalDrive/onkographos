# Ref-NeRF

Also called: Reflective Neural Radiance Fields, Structured View-Dependent Appearance for Neural Radiance Fields

## Overview
Ref-NeRF is a neural radiance field method that improves the representation of specular reflections by replacing NeRF’s parameterization of view-dependent outgoing radiance with a representation of reflected radiance, structured using spatially-varying scene properties [S2, S3]. It decomposes outgoing radiance into explicit diffuse and specular components, using the reflection of the viewing vector about the local normal vector as input to the directional network, which allows for smoother interpolation of glossy and reflective appearances compared to standard NeRF [S2, S5]. Unlike its siblings [DE-NeRF](8677-de-nerf.md) and [NeRFLight](9113-nerflight.md), Ref-NeRF does not disentangle incoming light from reflectance properties, meaning it cannot be used for relighting, but it enables intuitive material editing due to its structured output [S4, unofficial][S5].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinate $\mathbf{x}$, viewing direction $\mathbf{d}$, and surface normal $\mathbf{n}'$.
*   **Output:** Volume density $\sigma$, diffuse color $\mathbf{c}_d$, tint color $\mathbf{s}$, bottleneck feature vector $\mathbf{b}$, roughness scalar $\rho$, specular color $\mathbf{c}_s$, and final color $\mathbf{c}$.

**Data Structures:**
1.  **Spatial MLP:** A directionless Multi-Layer Perceptron.
    *   Input: Positional Encoded $\mathbf{x}$, $\gamma(\mathbf{x})$.
    *   Hidden Layers: $L$ layers with ReLU activations and skip connections from input to every 4th layer [S1, S3].
    *   Output: Density $\sigma$, diffuse color $\mathbf{c}_d$, tint color $\mathbf{s}$, bottleneck vector $\mathbf{b}$ (dimension 128), roughness $\rho$, and normal vector $\mathbf{n}'$ [S1, S3].
2.  **Directional MLP:** A Multi-Layer Perceptron.
    *   Input: Bottleneck vector $\mathbf{b}$, Integrated Directional Encoding (IDE) of reflection direction, and Fresnel term.
    *   Output: Specular color $\mathbf{c}_s$ [S1, S2].

**Numbered Steps:**

1.  **Positional Encoding:** Encode the 3D location $\mathbf{x}$ using standard positional encoding $\gamma(\mathbf{x})$ [S1, S3].
    $$
    \gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
    $$

2.  **Spatial Network Forward Pass:** Pass $\gamma(\mathbf{x})$ through the Spatial MLP to predict:
    *   Density $\sigma$
    *   Diffuse color $\mathbf{c}_d$
    *   Tint color $\mathbf{s}$
    *   Bottleneck feature vector $\mathbf{b} \in \mathbb{R}^{128}$
    *   Roughness scalar $\rho$
    *   Normal vector $\mathbf{n}'$ [S1, S3].

3.  **Normal Regularization:** Apply a volume density regularizer to concentrate density around surfaces and improve the quality of $\mathbf{n}'$, as NeRF's default geometry is "foggy" and normals are noisy [S2].

4.  **Reflection Vector Calculation:** Compute the reflection direction $\boldsymbol{\omega}_r$ of the viewing direction $\mathbf{d}$ about the normal $\mathbf{n}'$:
    $$
    \boldsymbol{\omega}_r = 2(\mathbf{d} \cdot \mathbf{n}')\mathbf{n}' - \mathbf{d}
    $$
    [S2, S5].

5.  **Integrated Directional Encoding (IDE):** Compute the IDE for the reflection direction $\boldsymbol{\omega}_r$ and roughness $\rho$. This involves convolving spherical harmonics with the von Mises-Fisher (vMF) distribution [S1].
    *   The IDE encodes the distribution of reflection directions towards the environment.
    *   It is analytically obtained from $\boldsymbol{\omega}_r$ and $\rho$ [S1].

6.  **Fresnel Term Calculation:** Compute the Fresnel term $F$ using the dot product of the normal $\mathbf{n}'$ and the viewing direction $\mathbf{d}$:
    $$
    F = \mathbf{n}' \cdot \mathbf{d}
    $$
    This accounts for Fresnel effects, where the angle between $\mathbf{n}'$ and $\mathbf{d}$ affects reflection strength [S1].

7.  **Directional Network Forward Pass:** Pass $\mathbf{b}$, the IDE of $\boldsymbol{\omega}_r$, and $F$ through the Directional MLP to predict the specular color $\mathbf{c}_s$ [S1, S2].

8.  **Color Composition:** Combine diffuse and specular components using the tint color $\mathbf{s}$ and a tone mapping function $\gamma(\cdot)$:
    $$
    \mathbf{c} = \gamma(\mathbf{c}_d + \mathbf{s} \odot \mathbf{c}_s)
    $$
    where $\odot$ denotes element-wise multiplication [S1].

9.  **Volume Rendering:** Render the pixel color $C(\mathbf{r})$ using standard volumetric rendering with samples $\mathbf{x}_i$, density $\sigma_i$, and color $\mathbf{c}_i$:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    [S1, S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class SpatialMLP(nn.Module):
    def __init__(self, input_dim, hidden_dim=256, num_layers=8, skip_indices=[4]):
        super().__init__()
        self.layers = nn.ModuleList()
        for i in range(num_layers):
            in_dim = input_dim if i == 0 else hidden_dim
            self.layers.append(nn.Linear(in_dim, hidden_dim))
        self.skip_indices = skip_indices
        self.output_layer = nn.Linear(hidden_dim, 128 + 3 + 3 + 1 + 3) # b, cd, s, rho, n'
        
    def forward(self, x):
        h = x
        for i, layer in enumerate(self.layers):
            h = torch.relu(layer(h))
            if i in self.skip_indices:
                h = torch.cat([h, x], dim=-1)
        out = self.output_layer(h)
        # Split outputs
        b = out[:, :128]
        cd = out[:, 128:131]
        s = out[:, 131:134]
        rho = out[:, 134:135]
        n_prime = out[:, 135:138]
        return b, cd, s, rho, n_prime

class DirectionalMLP(nn.Module):
    def __init__(self, input_dim, hidden_dim=256, num_layers=4):
        super().__init__()
        self.layers = nn.ModuleList()
        for i in range(num_layers):
            in_dim = input_dim if i == 0 else hidden_dim
            self.layers.append(nn.Linear(in_dim, hidden_dim))
        self.output_layer = nn.Linear(hidden_dim, 3) # cs
        
    def forward(self, x):
        h = x
        for layer in self.layers:
            h = torch.relu(layer(h))
        return self.output_layer(h)

def positional_encoding(x, L=10):
    # Standard NeRF positional encoding
    encodings = []
    for i in range(L):
        freq = 2**i * math.pi
        encodings.append(torch.sin(freq * x))
        encodings.append(torch.cos(freq * x))
    return torch.cat(encodings, dim=-1)

def integrated_directional_encoding(omega_r, rho):
    # Simplified placeholder for IDE calculation
    # In Ref-NeRF, this involves spherical harmonics convolved with vMF distribution
    # For implementation, this requires specific math libraries or closed-form approximations
    # Not given in the sources as a simple formula, but described as analytical [S1]
    # Returning a dummy vector for structure
    return torch.zeros(omega_r.shape[0], 128, device=omega_r.device)

def fresnel_term(n_prime, d):
    return torch.sum(n_prime * d, dim=-1, keepdim=True)

def tone_mapping(c):
    # Predefined tone mapping function gamma
    # Not given in the sources as a specific formula, assumed to be a standard sRGB or similar
    # [G] Using a simple gamma correction as a placeholder for gamma function
    return torch.pow(torch.clamp(c, 0, 1), 1/2.2)

def render_refnerf(x, d, spatial_mlp, directional_mlp):
    # 1. Positional Encoding
    x_enc = positional_encoding(x)
    
    # 2. Spatial Network
    b, cd, s, rho, n_prime = spatial_mlp(x_enc)
    
    # 3. Reflection Vector
    # omega_r = 2(d . n')n' - d
    dot_prod = torch.sum(d * n_prime, dim=-1, keepdim=True)
    omega_r = 2 * dot_prod * n_prime - d
    
    # 4. IDE and Fresnel
    ide = integrated_directional_encoding(omega_r, rho)
    f = fresnel_term(n_prime, d)
    
    # 5. Directional Network
    # Input to directional MLP: b, IDE, Fresnel
    # [G] Concatenating b, IDE, and Fresnel term. Dimensions must match.
    # Assuming IDE is 128-dim, b is 128-dim, f is 1-dim.
    dir_input = torch.cat([b, ide, f], dim=-1)
    cs = directional_mlp(dir_input)
    
    # 6. Color Composition
    # c = gamma(cd + s . cs)
    c_raw = cd + s * cs
    c = tone_mapping(c_raw)
    
    return c, cd, cs, rho, n_prime
```

**Parameters:**
*   Bottleneck feature vector $\mathbf{b}$: Dimensionality 128 [S1].
*   Number of frequency bands $L$: Not given in the sources.
*   Tone mapping function $\gamma$: Predefined, specific function not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Geometry Quality:** NeRF's default geometry is "foggy" and normals are noisy, making reflection directions inaccurate. A volume density regularizer is required to concentrate density around surfaces [S2].
*   **Specularity Misassignment:** Ref-NeRF may erroneously assign specularity to non-specular materials (e.g., bread slices) if geometric properties are not fully comprehended [S1].
*   **Performance:** The Integrated Directional Encoding is computationally expensive and slower than standard positional encoding [S5].
*   **Relighting:** Cannot be used for relighting as it does not disentangle incoming light from reflectance properties [S4, unofficial].

## Subtopics

### DE-NeRF
Decomposes NeRF into diffuse and environment components, allowing for relighting. Choose when relighting is required. [DE-NeRF](8677-de-nerf.md)

### NeRFLight
Explicitly models environment lighting and material properties. Choose when accurate relighting and material estimation are needed. [NeRFLight](9113-nerflight.md)

### Neural Light Field Representations
Represents light fields directly. Choose when high-fidelity light field capture is the goal. [Neural Light Field Representations](9273-neural-light-field-representations.md)

### Anisotropic Neural Representation Learning
Models anisotropic reflections. Choose when anisotropic materials (e.g., brushed metal) are present. [Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md)

### Convex Combination Color Decoder
Uses convex combinations for color decoding. Choose when a simpler color decomposition is preferred. [Convex Combination Color Decoder](11505-convex-combination-color-decoder.md)

### Unified Decompositional and Compositional NeRF Framework
Combines decompositional and compositional approaches. Choose for a unified framework handling various reflection types. [Unified Decompositional and Compositional NeRF Framework](14397-unified-decompositional-and-compositional-nerf-framework.md)

## Operating characteristics
*   **Time Complexity:** Computationally expensive due to the Integrated Directional Encoding (IDE) calculation, which is slower than standard positional encoding [S5].
*   **Memory Complexity:** Requires storing additional parameters for the spatial and directional MLPs, including the 128-dimensional bottleneck vector [S1].
*   **Measured Performance:** On the evaluated dataset, Ref-NeRF achieves 35.6dB PSNR and 11.5 degrees Mean Angular Error (MAE) for normals, compared to mip-NeRF's 30.3dB PSNR and 59.5 degrees MAE [S2].
*   **Strengths:** Significantly improves realism and accuracy of specular reflections on glossy surfaces; enables scene editing due to structured output radiation [S2, S5].
*   **Limitations:** Cannot be used for relighting; computationally expensive; may misassign specularity to non-specular materials [S1][S4, unofficial][S5].
*   **When to Use:** When rendering complex scenes with glossy/reflective surfaces and scene editing is desired, but relighting is not required.
*   **When Not to Use:** When relighting is required, or when computational resources are limited and standard NeRF suffices.

## Background
Ref-NeRF was introduced by Dor Verbin, Peter Hedman, Ben Mildenhall, Todd Zickler, Jonathan T. Barron, and Pratul P. Srinivasan from Harvard University and Google Research, published as arXiv:2112.03907v1 on Dec 7, 2021 [S2]. It was built on Mip-NeRF and designed to better model reflective surfaces by parameterizing NeRF radiance based on the reflection of the viewing direction about the local normal vector [S3].

## Sources
[S1] Specularity in NeRFs: A Comparative Study of Ref-NeRF and NRFF, https://doi.org/10.5201/ipol.2025.562, fetched 2026-10-26
[S2] Ref-NeRF: Structured View-Dependent Appearance for Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00541, fetched 2026-10-26
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S5] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
