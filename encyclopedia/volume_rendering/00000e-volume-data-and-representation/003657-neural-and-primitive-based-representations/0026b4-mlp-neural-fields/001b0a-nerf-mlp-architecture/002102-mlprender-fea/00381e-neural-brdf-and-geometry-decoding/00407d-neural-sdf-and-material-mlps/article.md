# Neural SDF and Material MLPs

Also called: IRON Pipeline Neural Architectures, IRON SDF and Material Networks

## Overview
This method defines the specific neural network architectures for decoding Signed Distance Functions (SDF) and spatially-varying material properties (diffuse albedo, specular albedo, roughness) within the IRON pipeline [S1]. It solves the problem of representing continuous geometry and appearance from photometric images by using distinct Multi-Layer Perceptrons (MLPs) for each property, addressing the spectral bias of MLPs via positional encoding [S1]. It differs from sibling methods like [TensoRF Feature Decoding Architecture](8181-tensorf-feature-decoding-architecture.md) by using explicit, separate MLPs for SDF and material components rather than tensor-decoded features, and from [Neural Surface Rendering](9960-neural-surface-rendering.md) by focusing on the specific architectural hyperparameters for inverse rendering optimization [S1].

## How it works
**Inputs and Outputs:**
*   **SDF MLP:** Input is a 3D location $x \in \mathbb{R}^3$. Output is the signed distance $S$ and feature vector $f$.
*   **Material MLPs:** Inputs are 3D location $x$, surface normal $n$, and potentially feature vectors $f$. Outputs are diffuse albedo $\beta$, specular albedo $\kappa$, and roughness $\alpha$.
*   **vMF Material MLP (Variant):** Input is surface point $x_s$. Output is roughness $\rho \in [0, 1]$, diffuse color $C_d \in [0, 1]$, and specular tint $\alpha \in [0, 1]$ [S2].

**Data Structures:**
All networks are MLPs with ReLU activations (implied by standard practice, though [S1] specifies skip connections and [S2] specifies softplus for vMF variant).
*   **SDF MLP:** 8 layers, width 256. Skip connection at layer 4.
*   **Diffuse Albedo MLP:** 8 layers, width 256. Skip connection at layer 4.
*   **Specular Albedo MLP:** 4 layers, width 256.
*   **Roughness MLP:** 4 layers, width 256.

**Positional Encoding:**
Input coordinates are encoded using positional encoding functions $\gamma(x)$ to compensate for spectral bias [S1].
*   SDF Input $x$: 6 frequencies.
*   Diffuse Albedo Input $x$: 10 frequencies.
*   Diffuse Albedo Input $n$: 4 frequencies.
*   Specular Albedo Input $x$: 6 frequencies.
*   Roughness Input $x$: 6 frequencies.

**Numbered Steps:**
1.  **Encode Input:** Apply positional encoding to the input 3D location $x$ (and normal $n$ if applicable) based on the specific MLP's frequency requirements [S1].
2.  **Forward Pass (SDF):** Pass the encoded $x$ through the 8-layer SDF MLP. The output is the signed distance $S$ and a feature vector $f$ [S1].
3.  **Forward Pass (Materials):**
    *   For diffuse albedo $\beta$: Pass encoded $x$ (10 freq) and encoded $n$ (4 freq) through the 8-layer Diffuse Albedo MLP [S1].
    *   For specular albedo $\kappa$: Pass encoded $x$ (6 freq) through the 4-layer Specular Albedo MLP [S1].
    *   For roughness $\alpha$: Pass encoded $x$ (6 freq) through the 4-layer Roughness MLP [S1].
4.  **vMF Variant (Alternative Material Parameterization):**
    *   Pass surface point $x_s$ through a material MLP $M$ with softplus activation to output $\rho, C_d, \alpha$ [S2].
    *   Compute reflection direction $l_r = 2(-d \cdot n)n + d$, where $d$ is the view direction and $n$ is the normal [S2].
    *   Approximate BRDF using von Mises-Fisher distribution: $f(l, -d, x_s) \approx G(l, l_r, x_s) = \alpha \exp(\rho(x_s)(l \cdot l_r - 1))$ [S2].

**Formulas:**
*   **vMF BRDF Approximation:**
    $$f(l, -d, x_s) \approx \alpha \exp(\rho(x_s)(l \cdot l_r - 1))$$
    Where $l$ is the light direction, $d$ is the view direction, $n$ is the normal, $l_r$ is the reflection direction, $\alpha$ is specular tint, and $\rho$ is roughness [S2].

**Parameters:**
*   SDF MLP: 8 layers, width 256, skip at layer 4, 6 PE frequencies for $x$ [S1].
*   Diffuse Albedo MLP: 8 layers, width 256, skip at layer 4, 10 PE frequencies for $x$, 4 PE frequencies for $n$ [S1].
*   Specular Albedo MLP: 4 layers, width 256, 6 PE frequencies for $x$ [S1].
*   Roughness MLP: 4 layers, width 256, 6 PE frequencies for $x$ [S1].
*   vMF Material MLP: Softplus activation [S2].

**Edge Cases and Pitfalls:**
*   **Spectral Bias:** Without positional encoding, MLPs struggle to learn high-frequency details [S1].
*   **Normal Consistency:** In the IRON pipeline, the "second surface normal" used in diffuse albedo decoding equals the viewing direction in the first stage of volumetric radiance fields rendering [S1].
*   **BRDF Symmetry:** The vMF approximation assumes rotational symmetry with respect to the reflection direction $l_r$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

def positional_encoding(x, num_frequencies):
    """
    Standard positional encoding for neural fields.
    x: tensor of shape (..., 3)
    num_frequencies: int
    Returns: tensor of shape (..., 3 * 2 * num_frequencies)
    """
    # Implementation of standard PE: sin(2^i * pi * x) and cos(2^i * pi * x)
    # This is a standard math operation [G]
    frequencies = torch.arange(num_frequencies, device=x.device).float()
    scales = 2 ** frequencies
    scaled_x = x.unsqueeze(-1) * scales * math.pi
    encoded = torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-1)
    return encoded.view(x.shape[0], -1)

class SkipMLP(nn.Module):
    def __init__(self, input_dim, output_dim, num_layers, width, skip_layer):
        super().__init__()
        self.layers = nn.ModuleList()
        self.skip_layer = skip_layer
        
        # Layer 0
        self.layers.append(nn.Linear(input_dim, width))
        
        # Hidden layers
        for _ in range(1, num_layers):
            self.layers.append(nn.Linear(width, width))
            
        # Output layer
        self.layers.append(nn.Linear(width, output_dim))

    def forward(self, x):
        h = x
        for i, layer in enumerate(self.layers):
            if i == 0:
                h = layer(h)
            elif i == self.skip_layer:
                # Skip connection: concatenate input to hidden representation
                h = torch.cat([h, x], dim=-1)
                h = layer(h)
            else:
                h = layer(h)
            if i < len(self.layers) - 1:
                h = torch.relu(h)
        return h

class NeuralSDF:
    def __init__(self):
        # 8 layers, width 256, skip at 4th layer (index 4 in 0-based list of hidden+output? 
        # Source says "skip connection at the 4th layer". In IRON paper context, 
        # this usually means the 4th hidden layer or the layer where skip is added.
        # Assuming standard NeRF-like structure: layers 0-7. Skip at layer 4.
        self.mlp = SkipMLP(input_dim=3*2*6, output_dim=1, num_layers=8, width=256, skip_layer=4)

    def forward(self, x):
        x_enc = positional_encoding(x, 6)
        s = self.mlp(x_enc)
        return s

class NeuralMaterial:
    def __init__(self):
        # Diffuse: 8 layers, width 256, skip at 4. Input x (10 freq), n (4 freq)
        self.diffuse_mlp = SkipMLP(input_dim=3*2*10 + 3*2*4, output_dim=3, num_layers=8, width=256, skip_layer=4)
        # Specular: 4 layers, width 256. Input x (6 freq)
        self.specular_mlp = SkipMLP(input_dim=3*2*6, output_dim=3, num_layers=4, width=256, skip_layer=None)
        # Roughness: 4 layers, width 256. Input x (6 freq)
        self.roughness_mlp = SkipMLP(input_dim=3*2*6, output_dim=1, num_layers=4, width=256, skip_layer=None)

    def forward(self, x, n):
        x_enc_diff = positional_encoding(x, 10)
        n_enc_diff = positional_encoding(n, 4)
        diffuse = self.diffuse_mlp(torch.cat([x_enc_diff, n_enc_diff], dim=-1))
        
        x_enc_spec = positional_encoding(x, 6)
        specular = self.specular_mlp(x_enc_spec)
        roughness = self.roughness_mlp(x_enc_spec)
        
        return diffuse, specular, roughness

class VMFMaterial:
    """
    Variant from [S2]. Uses softplus activation.
    """
    def __init__(self):
        # Material MLP M: xs -> (rho, Cd, alpha)
        # Input dim depends on xs. Assuming xs is 3D location + normal? 
        # Source says "surface point xs". Let's assume 3D location for simplicity 
        # as per standard MLP input, but note source doesn't specify PE for this variant explicitly.
        # We assume standard PE or raw input. Source [S2] doesn't specify PE for M.
        # We will use raw input for M as per [S2] description "M : xs -> ...".
        self.mlp = nn.Sequential(
            nn.Linear(3, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 3) # rho, Cd, alpha
        )
        # Note: Source [S2] specifies softplus activation for output.
        self.softplus = nn.Softplus()

    def forward(self, xs):
        params = self.mlp(xs)
        rho, Cd, alpha = self.softplus(params).split(1, dim=-1)
        return rho, Cd, alpha

    def compute_brdf(self, l, d, n, xs):
        """
        Approximate BRDF using vMF distribution [S2].
        l: light direction
        d: view direction
        n: normal
        xs: surface point
        """
        rho, Cd, alpha = self.forward(xs)
        # Compute reflection direction lr = 2(-d . n)n + d
        dot_dn = torch.sum(d * n, dim=-1, keepdim=True)
        lr = 2 * (-dot_dn) * n + d
        lr = lr / torch.norm(lr, dim=-1, keepdim=True) # Normalize
        
        # l . lr
        dot_lr = torch.sum(l * lr, dim=-1, keepdim=True)
        
        # G(l, lr, xs) = alpha * exp(rho * (l . lr - 1))
        brdf = alpha * torch.exp(rho * (dot_lr - 1))
        return brdf
```

## Subtopics

### Material Parameterization with vMF BRDF
This variant, used in NeIF [S2], parameterizes spatially-varying material as roughness $\rho$, diffuse color $C_d$, and specular tint $\alpha$ using a material MLP with softplus activation. It approximates the BRDF as a von Mises-Fisher distribution, assuming rotational symmetry around the reflection direction. Choose this when modeling directional emitted rays and environment occlusions in static 3D environments with a focus on efficient BRDF approximation [S2].

## Operating characteristics
*   **Time Complexity:** Inference is $O(L \cdot W^2)$ per MLP, where $L$ is layers and $W$ is width. Multiple MLPs are called per point.
*   **Memory Complexity:** $O(L \cdot W^2)$ parameters per MLP. Total memory is sum of all MLPs.
*   **Performance:** Trained for 100k iterations with 512 randomly sampled pixels per iteration using $\ell_1$ image loss and eikonal regularization ($\lambda_1 = 0.1$) [S1].
*   **Strengths:** Explicit control over geometry and material properties; handles spectral bias via PE [S1].
*   **Limitations:** Requires separate MLPs for each property, increasing parameter count compared to shared feature decoders [S1].
*   **When to use:** Inverse rendering from photometric images where explicit SDF and material separation is beneficial [S1].
*   **When not to use:** When memory is extremely constrained and shared feature decoding is preferred [S1].

## Background
Originated in the IRON (Inverse Rendering by Optimizing Neural SDFs and Materials) pipeline [S1]. Supported by NSF (IIS-2008313), Intel, and AWS [S1]. The vMF variant appears in NeIF (Neural Illumination Fields) [S2].

## Sources
[S1] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
[S2] A Pre-convolved Representation for Plug-and-Play Neural Illumination Fields, https://doi.org/10.1609/aaai.v38i7.28618, fetched 2026-10-29
