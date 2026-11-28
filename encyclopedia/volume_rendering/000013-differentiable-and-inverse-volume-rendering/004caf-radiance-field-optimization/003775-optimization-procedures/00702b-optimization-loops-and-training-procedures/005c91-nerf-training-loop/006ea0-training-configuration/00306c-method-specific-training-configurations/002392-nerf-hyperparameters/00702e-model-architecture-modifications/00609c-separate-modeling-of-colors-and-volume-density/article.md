# Separate Modeling of Colors and Volume Density

Also called: Color Branch and Density Branch, Dual-Branch NeRF, Separate MLP for Color and Density

## Overview
This method addresses the overfitting and artifact issues in few-shot view synthesis by decoupling the prediction of volume density (geometry) and color (appearance) into two distinct neural networks. Unlike the baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) which uses a single monolithic MLP to predict both outputs synchronously, this approach employs separate branches with different positional encoding frequencies, prioritizing low-frequency information for geometry and high-frequency information for appearance.

## How it works
The method replaces the single output head of the baseline NeRF with two specialized Multi-Layer Perceptrons (MLPs): a Density Branch $D_\theta$ and a Color Branch $C_\theta$. The core innovation is the use of distinct positional encoding frequencies for the inputs of each branch, based on the assumption that geometry is smoother (lower frequency) than appearance (higher frequency) [S1].

### Inputs and Outputs
*   **Inputs:**
    *   $\mathbf{x} \in \mathbb{R}^3$: 3D spatial coordinate.
    *   $\mathbf{d} \in \mathbb{R}^3$: Viewing direction vector.
*   **Outputs:**
    *   $\sigma \in \mathbb{R}$: Volume density.
    *   $\mathbf{c} \in \mathbb{R}^3$: View-dependent color.

### Data Structures and Parameters
The network parameters $\theta$ are split into $\theta_D$ for the Density Branch and $\theta_C$ for the Color Branch.

**Hyperparameters:**
*   $L_1, L_2, L_3$: Integers controlling the number of frequency bands in the positional encoding.
*   Constraint: $L_3 \le L_1 \le L_2$ [S1].
    *   $L_1$: Frequency for Density Branch spatial input.
    *   $L_2$: Frequency for Color Branch spatial input.
    *   $L_3$: Frequency for Color Branch directional input.
*   $\gamma_L(\mathbf{v})$: Positional encoding function applied to vector $\mathbf{v}$ with $L$ frequency bands [S1].

### Network Topology and Formulas

**1. Density Branch ($D_\theta$)**
The Density Branch estimates $\sigma$ using only the spatial coordinate $\mathbf{x}$ with low-frequency encoding $L_1$. It incorporates inputs into every layer (similar to mi-MLP) to improve few-shot performance [S1].

For layer $i$ (where $i \ge 1$):
$$
f_1^D = \phi_1^D(\gamma_{L_1}(\mathbf{x}))
$$
$$
f_i^D = \phi_i^D(f_{i-1}^D, \gamma_{L_1}(\mathbf{x})) \quad \text{for } i \ge 2
$$
Where:
*   $\phi_i^D$ is the $i$-th layer of the Density Branch MLP (linear transformation + activation).
*   $f_i^D$ is the output feature of layer $i$.
*   The final volume density is $\sigma = f_{N_D}^D$ (assuming $N_D$ layers).

**2. Color Branch ($C_\theta$)**
The Color Branch estimates $\mathbf{c}$ using the spatial coordinate $\mathbf{x}$ with high-frequency encoding $L_2$ and the viewing direction $\mathbf{d}$ with frequency $L_3$. It includes a residual interaction mechanism with the Density Branch features to aid geometry recovery [S1].

For layer $i$ (where $i \ge 1$):
$$
f_1^C = \phi_1^C(\gamma_{L_2}(\mathbf{x}))
$$
For $i \ge 2$:
$$
f_{i-1}^C = \phi_{i-1}^C(f_{i-2}^C, \gamma_{L_3}(\mathbf{d})) + f_{i-1}^C
$$
$$
f_i^C = \phi_i^C(f_{i-1}^C, \gamma_{L_3}(\mathbf{d}))
$$
*Note: The source formula for the interaction step $f_{i-1}^C = \dots + f_{i-1}^C$ implies a residual connection where the previous layer's output is added to the result of the current layer's transformation. The exact implementation of the "interaction" described as "beneficial to better geometry recovery" is defined by these equations [S1].*
Where:
*   $\phi_i^C$ is the $i$-th layer of the Color Branch MLP.
*   $f_i^C$ is the output feature of layer $i$.
*   The final color is $\mathbf{c} = f_{N_C}^C$ (assuming $N_C$ layers).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn
import torch.nn.functional as F

class PositionalEncoding(nn.Module):
    """
    Standard positional encoding with L frequency bands.
    Input v: (..., 3)
    Output: (..., 6*L)
    """
    def __init__(self, L):
        super().__init__()
        self.L = L

    def forward(self, v):
        # v shape: (..., 3)
        # Create frequencies
        # freq_bands = 2^0, 2^1, ..., 2^(L-1)
        freq_bands = torch.tensor([2.0 ** i for i in range(self.L)], device=v.device, dtype=v.dtype)
        # Expand v to match freq_bands shape for broadcasting
        # v: (..., 3) -> (..., 1, 3)
        # freq_bands: (L,) -> (1, L, 1)
        # result: (..., L, 3)
        scaled_v = v.unsqueeze(-2) * freq_bands.unsqueeze(0).unsqueeze(-1)
        # Apply sin and cos
        sin_vals = torch.sin(scaled_v)
        cos_vals = torch.cos(scaled_v)
        # Concatenate along the last dimension
        # (..., L, 3) -> (..., L*3)
        encoded = torch.cat([sin_vals, cos_vals], dim=-1)
        return encoded

class DensityBranch(nn.Module):
    def __init__(self, in_dim, hidden_dim, out_dim, num_layers, L1):
        super().__init__()
        self.L1 = L1
        self.pe = PositionalEncoding(L1)
        self.layers = nn.ModuleList()
        
        # First layer
        self.layers.append(nn.Linear(in_dim + self.pe.L * 6, hidden_dim))
        self.layers.append(nn.ReLU())
        
        # Hidden layers
        for _ in range(num_layers - 2):
            self.layers.append(nn.Linear(hidden_dim + self.pe.L * 6, hidden_dim))
            self.layers.append(nn.ReLU())
            
        # Output layer
        self.layers.append(nn.Linear(hidden_dim + self.pe.L * 6, out_dim))

    def forward(self, x):
        # x: (..., 3)
        pe_x = self.pe(x)
        # Concatenate original x and PE? 
        # Source says input is gamma_L1(x). Usually PE replaces or augments.
        # NeRF standard PE augments. Let's assume augmentation for compatibility with standard NeRF pipelines mentioned in S1.
        # However, S1 says "reduce dimensions of input embeddings". 
        # Let's assume the input to the MLP is the PE vector itself or PE + x. 
        # Standard NeRF uses PE. S1 says "dimensions of encoded input embeddings... decide how detailed".
        # We will use the PE vector as the primary feature, potentially concatenated with x if needed, 
        # but strictly following S1 formula: input is gamma_L1(x).
        
        h = pe_x
        for i, layer in enumerate(self.layers):
            if i % 2 == 0: # Linear layer
                # Concatenate input x (or its PE) at every layer
                # S1: "incorporate inputs into each layer"
                # Input to layer i is (prev_h, gamma_L1(x))
                h = torch.cat([h, pe_x], dim=-1)
                h = layer(h)
            else: # Activation
                h = layer(h)
        return h

class ColorBranch(nn.Module):
    def __init__(self, in_dim_x, in_dim_d, hidden_dim, out_dim, num_layers, L2, L3):
        super().__init__()
        self.L2 = L2
        self.L3 = L3
        self.pe_x = PositionalEncoding(L2)
        self.pe_d = PositionalEncoding(L3)
        self.layers = nn.ModuleList()
        
        # First layer takes only spatial input
        self.layers.append(nn.Linear(in_dim_x + self.pe_x.L * 6, hidden_dim))
        self.layers.append(nn.ReLU())
        
        # Hidden layers take previous feature and direction
        for _ in range(num_layers - 2):
            self.layers.append(nn.Linear(hidden_dim + self.pe_d.L * 6, hidden_dim))
            self.layers.append(nn.ReLU())
            
        # Output layer
        self.layers.append(nn.Linear(hidden_dim + self.pe_d.L * 6, out_dim))

    def forward(self, x, d):
        pe_x = self.pe_x(x)
        pe_d = self.pe_d(d)
        
        h = pe_x
        for i, layer in enumerate(self.layers):
            if i == 0:
                # First layer: input is gamma_L2(x)
                h = layer(h)
            else:
                # Subsequent layers: interaction with density branch features is mentioned in text,
                # but formula 10 shows interaction between color branch layers themselves via residual.
                # Formula: f_{i-1}^C = phi(f_{i-2}, d) + f_{i-1}
                # This implies a residual connection.
                
                # Calculate new feature from previous feature and direction
                h_input = torch.cat([h, pe_d], dim=-1)
                h_new = layer(h_input)
                
                # Residual connection: add previous h to new h
                # Note: S1 formula: f_{i-1}^C = phi(...) + f_{i-1}^C
                # This notation is slightly ambiguous. It likely means:
                # temp = phi(prev, d)
                # curr = temp + prev
                # But the variable names in S1 are f_{i-1} on both sides.
                # Let's interpret as: h = layer(h, d) + h
                h = h_new + h
                
        return h

class SeparateModel(nn.Module):
    def __init__(self, hidden_dim, num_layers_density, num_layers_color, L1, L2, L3):
        super().__init__()
        # Assuming input dim 3 for x and d
        self.density_branch = DensityBranch(3, hidden_dim, 1, num_layers_density, L1)
        self.color_branch = ColorBranch(3, 3, hidden_dim, 3, num_layers_color, L2, L3)
        
    def forward(self, x, d):
        # x: (N, 3), d: (N, 3)
        sigma = self.density_branch(x)
        c = self.color_branch(x, d)
        return sigma, c
```

### Edge Cases and Pitfalls
*   **Frequency Selection:** The constraint $L_3 \le L_1 \le L_2$ is critical. If $L_1$ is too high, the density branch may overfit to noise, creating artifacts. If $L_2$ is too low, fine appearance details are lost [S1].
*   **Interaction Implementation:** The residual interaction in the Color Branch ($f_{i-1}^C = \dots + f_{i-1}^C$) must be implemented carefully to ensure gradient flow. The source notes this interaction is "beneficial to better geometry recovery" [S1].
*   **Few-Shot Overfitting:** While this method reduces artifacts, it is specifically designed for few-shot scenarios. In dense-view settings, the standard shared MLP might still be competitive or superior due to parameter efficiency [S1].

## Operating characteristics
*   **Complexity:** The method increases the number of parameters compared to a single shared MLP because two separate networks are trained. However, the Density Branch uses lower-dimensional embeddings ($L_1$), partially offsetting the cost.
*   **Performance:** It significantly reduces artifacts in few-shot view synthesis compared to vanilla NeRF [S1].
*   **Strengths:** Better geometry recovery in object-centric scenes; reduced overfitting in few-shot settings.
*   **Limitations:** Requires careful tuning of $L_1, L_2, L_3$. May not provide significant benefits in dense-view scenarios where the shared MLP is sufficient.

## Background
This method was introduced in the paper "Is Vanilla MLP in Neural Radiance Field Enough for Few-Shot View Synthesis?" (CVPR 2024) [S1]. It was developed to address the limitations of the standard NeRF architecture when applied to few-shot view synthesis, where the shared MLP tends to overfit and produce geometric artifacts. The design is motivated by the observation that geometry is typically smoother than appearance [S1].

## Sources
[S1] Is Vanilla MLP in Neural Radiance Field Enough for Few-Shot View Synthesis?, https://doi.org/10.1109/cvpr52733.2024.01918, fetched 2026-10-26
