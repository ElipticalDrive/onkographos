# Separate Modeling of Colors and Volume Density

Also called: Color Branch and Density Branch, Dual-Branch NeRF, Separate MLP for Color and Density

## Overview
This method addresses the overfitting and artifact issues in NeRF-based view synthesis by decoupling the prediction of volume density (geometry) and color (appearance) into two distinct neural network branches. Unlike the standard NeRF architecture which uses a single shared MLP to predict both outputs synchronously from the same input embeddings, this approach employs separate MLPs with different positional encoding frequencies. This separation allows the model to prioritize low-frequency information for geometry, which is typically smoother, while retaining high-frequency details for appearance, thereby improving geometry recovery and reducing artifacts [S1].

## How it works
The method replaces the single shared MLP of the baseline NeRF architecture [Parent](11313-model-architecture-modifications.md) with two specialized branches: a Density Branch and a Color Branch. The core innovation is the use of distinct positional encoding frequencies for each branch to reflect the physical prior that geometry is less detailed than appearance.

### Inputs and Outputs
*   **Inputs**:
    *   $\mathbf{x} \in \mathbb{R}^3$: 3D spatial coordinate.
    *   $\mathbf{d} \in \mathbb{R}^3$: Viewing direction vector.
*   **Outputs**:
    *   $\sigma$: Volume density scalar.
    *   $\mathbf{c} \in \mathbb{R}^3$: View-dependent color vector.

### Data Structures and Parameters
The network consists of two MLPs: $D_\theta$ (Density Branch) and $C_\theta$ (Color Branch). The key hyperparameters are the positional encoding frequencies $L_1, L_2, L_3$, which control the dimensionality and frequency of the input embeddings.

*   **Frequency Hierarchy**: The frequencies must satisfy the constraint $L_3 \le L_1 \le L_2$ [S1].
    *   $L_1$: Frequency for the Density Branch position input.
    *   $L_2$: Frequency for the Color Branch position input.
    *   $L_3$: Frequency for the Color Branch direction input.
*   **Positional Encoding**: $\gamma_L(\mathbf{v})$ denotes the standard positional encoding function applied to vector $\mathbf{v}$ with frequency parameter $L$ [S1].

### Algorithm Steps

1.  **Density Branch Forward Pass**:
    The Density Branch $D_\theta$ takes only the position $\mathbf{x}$ encoded with frequency $L_1$. It incorporates the input into every layer to facilitate feature reuse [S1].
    *   Initialize the first layer output:
        $$f_1^D = \phi_1^D(\gamma_{L_1}(\mathbf{x}))$$
    *   For subsequent layers $i = 2, \dots, N_D$:
        $$f_i^D = \phi_i^D(f_{i-1}^D, \gamma_{L_1}(\mathbf{x}))$$
    *   The final volume density is derived from the last layer's output (implementation detail: typically a linear projection or direct output of the final layer feature) [S1].

2.  **Color Branch Forward Pass**:
    The Color Branch $C_\theta$ takes the position $\mathbf{x}$ encoded with frequency $L_2$ and the direction $\mathbf{d}$ encoded with frequency $L_3$. It also incorporates inputs into each layer and includes a residual interaction with the previous layer's output [S1].
    *   Initialize the first layer output:
        $$f_1^C = \phi_1^C(\gamma_{L_2}(\mathbf{x}))$$
    *   For subsequent layers $i = 2, \dots, N_C$:
        $$f_{i-1}^C = \phi_{i-1}^C(f_{i-2}^C, \gamma_{L_3}(\mathbf{d})) + f_{i-1}^C$$
        $$f_i^C = \phi_i^C(f_{i-1}^C, \gamma_{L_3}(\mathbf{d}))$$
    *   The final color $\mathbf{c}$ is derived from the last layer's output [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class PositionalEncoding(nn.Module):
    """
    Standard positional encoding for NeRF.
    """
    def __init__(self, num_freqs):
        super().__init__()
        self.num_freqs = num_freqs

    def forward(self, x):
        # x shape: (..., 3)
        # Standard NeRF PE: sin(2^i * pi * x) for i in range(num_freqs)
        # Implementation omitted for brevity, assumed to return shape (..., 3 * 2 * num_freqs)
        # [G] Standard implementation:
        scales = torch.tensor([2**i for i in range(self.num_freqs)], dtype=x.dtype, device=x.device)
        shaped = x.shape[:-1] + (-1,)
        freq_bands = scales.view(1, -1) * x.view(-1, 1)
        freq_bands = freq_bands.reshape(shaped + (self.num_freqs, 3))
        return torch.cat([torch.sin(freq_bands), torch.cos(freq_bands)], dim=-1)

class DensityBranch(nn.Module):
    def __init__(self, input_dim, hidden_dim, num_layers, freq_x):
        super().__init__()
        self.pe = PositionalEncoding(freq_x)
        self.pe_dim = input_dim * 2 * freq_x # Assuming input_dim=3 for x
        
        # Layers
        self.layers = nn.ModuleList()
        # Layer 1
        self.layers.append(nn.Linear(self.pe_dim + self.pe_dim, hidden_dim)) # Input + PE
        # Layers 2..N
        for _ in range(num_layers - 1):
            self.layers.append(nn.Linear(hidden_dim + self.pe_dim, hidden_dim))
            
        self.relu = nn.ReLU()

    def forward(self, x):
        pe_x = self.pe(x)
        h = self.layers[0](torch.cat([pe_x, pe_x], dim=-1))
        h = self.relu(h)
        
        for i in range(1, len(self.layers)):
            # Incorporate input into each layer
            h = torch.cat([h, pe_x], dim=-1)
            h = self.layers[i](h)
            h = self.relu(h)
            
        return h # Feature vector, density sigma is typically a linear projection of this

class ColorBranch(nn.Module):
    def __init__(self, input_dim_pos, input_dim_dir, hidden_dim, num_layers, freq_x, freq_d):
        super().__init__()
        self.pe_x = PositionalEncoding(freq_x)
        self.pe_d = PositionalEncoding(freq_d)
        
        self.pe_x_dim = input_dim_pos * 2 * freq_x
        self.pe_d_dim = input_dim_dir * 2 * freq_d
        
        self.layers = nn.ModuleList()
        # Layer 1: takes only position PE
        self.layers.append(nn.Linear(self.pe_x_dim, hidden_dim))
        
        # Layers 2..N: take previous hidden + direction PE
        for _ in range(num_layers - 1):
            self.layers.append(nn.Linear(hidden_dim + self.pe_d_dim, hidden_dim))
            
        self.relu = nn.ReLU()

    def forward(self, x, d):
        pe_x = self.pe_x(x)
        pe_d = self.pe_d(d)
        
        # Layer 1
        h = self.layers[0](pe_x)
        h = self.relu(h)
        
        for i in range(1, len(self.layers)):
            # Interaction: add previous layer output to the result of linear + direction PE
            # Formula: f_{i-1}^C = phi_{i-1}^C(f_{i-2}^C, gamma(d)) + f_{i-1}^C
            # Note: The source formula indices are slightly ambiguous regarding the residual addition.
            # Interpreting f_{i-1}^C as the output of layer i-1.
            # The term phi_{i-1}^C(...) computes the transformation.
            # The residual adds the previous output f_{i-1}^C.
            
            # Let prev_h be the output of layer i-1 (which is h at start of loop iteration i)
            # The source defines:
            # f_{i-1}^C = phi_{i-1}^C(f_{i-2}^C, gamma(d)) + f_{i-1}^C
            # f_i^C = phi_i^C(f_{i-1}^C, gamma(d))
            
            # In loop index i (starting 1 for 2nd layer):
            # We are computing layer i.
            # Input to layer i is f_{i-1}^C.
            # But f_{i-1}^C itself was defined as a residual sum.
            
            # Let's trace carefully:
            # i=1 (Layer 2 in 0-indexed list, or 2nd layer overall):
            # f_1^C is output of Layer 1.
            # f_1^C (new) = phi_1^C(f_0^C, pe_d) + f_1^C (old) ? 
            # The source says: f_{i-1}^C = phi_{i-1}^C(...) + f_{i-1}^C.
            # This implies the residual is added to the *output* of the previous layer's transformation?
            # Or is it a skip connection?
            # "f_{i-1}^C = phi_{i-1}^C(f_{i-2}^C, gamma_{L3}(d)) + f_{i-1}^C"
            # This looks like: New_Hidden = Linear(Prev_Hidden, Dir) + Prev_Hidden.
            
            prev_h = h
            linear_out = self.layers[i](torch.cat([prev_h, pe_d], dim=-1))
            h = linear_out + prev_h
            h = self.relu(h)
            
        return h

class SeparateModel(nn.Module):
    def __init__(self, hidden_dim, num_layers, freq_x_density, freq_x_color, freq_d_color):
        super().__init__()
        self.density_branch = DensityBranch(
            input_dim=3, 
            hidden_dim=hidden_dim, 
            num_layers=num_layers, 
            freq_x=freq_x_density
        )
        self.color_branch = ColorBranch(
            input_dim_pos=3, 
            input_dim_dir=3, 
            hidden_dim=hidden_dim, 
            num_layers=num_layers, 
            freq_x=freq_x_color, 
            freq_d=freq_d_color
        )
        # Final projections [G]
        self.sigma_proj = nn.Linear(hidden_dim, 1)
        self.color_proj = nn.Linear(hidden_dim, 3)

    def forward(self, x, d):
        # Density Branch
        # f_N^D is the output of the last layer
        # We need to handle the layer indexing correctly.
        # Density Branch loop runs for layers 1 to N.
        # The output of the last layer is the feature.
        dens_feat = self.density_branch(x)
        sigma = self.sigma_proj(dens_feat).squeeze(-1)
        
        # Color Branch
        col_feat = self.color_branch(x, d)
        color = self.color_proj(col_feat)
        
        return sigma, color
```

### Edge Cases and Pitfalls
*   **Frequency Selection**: The constraint $L_3 \le L_1 \le L_2$ is critical. If $L_1$ is too high, the density branch may overfit to high-frequency noise, defeating the purpose of smoothing geometry. If $L_2$ is too low, color details may be lost [S1].
*   **Residual Interaction**: The specific residual formulation in the Color Branch ($f_{i-1}^C = \phi_{i-1}^C(...) + f_{i-1}^C$) is an empirical finding. Incorrect implementation of this residual connection may break the beneficial interaction between branches [S1].
*   **Background Artifacts**: In object-centric scenes, background artifacts may persist. The source notes a separate regularization term and sampling strategy (sampling pixels outside the image space) are used in conjunction with this architecture to mitigate this, but the architecture itself does not solve it [S1].

## Operating characteristics
*   **Complexity**: The method increases the number of parameters compared to a single shared MLP because it maintains two separate networks. However, the Density Branch uses lower-dimensional embeddings ($L_1$), partially offsetting the cost.
*   **Performance**: The method is designed to reduce artifacts and improve geometry recovery, particularly in few-shot view synthesis scenarios where standard NeRF overfits [S1].
*   **Strengths**: Better geometry recovery due to low-frequency prioritization for density; plug-and-play compatibility with existing NeRF pipelines [S1].
*   **Limitations**: Requires careful tuning of frequency hyperparameters ($L_1, L_2, L_3$); may still suffer from background artifacts in object-centric scenes without additional regularization [S1].

## Background
This method was introduced in the paper "Is Vanilla MLP in Neural Radiance Field Enough for Few-Shot View Synthesis?" [S1]. It was developed to address the limitations of the standard NeRF architecture, specifically the tendency to overfit and produce artifacts when trained on limited data (few-shot settings). The motivation stems from the observation that geometry is typically smoother than appearance, justifying the separate modeling with different frequency encodings [S1].

## Sources
[S1] Is Vanilla MLP in Neural Radiance Field Enough for Few-Shot View Synthesis?, https://doi.org/10.1109/cvpr52733.2024.01918, fetched 2026-10-26
