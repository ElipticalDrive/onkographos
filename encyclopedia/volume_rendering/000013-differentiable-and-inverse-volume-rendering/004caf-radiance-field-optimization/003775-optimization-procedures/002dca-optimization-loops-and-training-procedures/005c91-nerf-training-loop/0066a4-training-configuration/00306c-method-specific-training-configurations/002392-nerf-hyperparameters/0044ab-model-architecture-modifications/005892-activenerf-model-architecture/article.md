# ActiveNeRF Model Architecture

Also called: ActiveNeRF MLP structure, Uncertainty-aware NeRF network

## Overview
ActiveNeRF extends the standard NeRF architecture [S1] by incorporating uncertainty estimation into the radiance field representation. While standard NeRF predicts volume density $\sigma$ and view-dependent color $\mathbf{c}$, ActiveNeRF additionally predicts a prediction variance (uncertainty) for each point along the ray. This modification allows the model to quantify its confidence in the rendered output, which is critical for active view selection and robust scene reconstruction. It differs from siblings like [DoF-NeRF](9432-dof-nerf.md) or [Aleth-NeRF](13764-aleth-nerf.md) by explicitly modeling the predictive variance of the MLP outputs as a distinct scalar field alongside density and color, rather than relying solely on ensemble methods or auxiliary geometric constraints for uncertainty.

## How it works
The ActiveNeRF model $F_\Theta$ maps a 3D spatial position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ to three outputs: volume density $\sigma \in \mathbb{R}$, RGB color $\mathbf{c} \in \mathbb{R}^3$, and prediction variance $\nu \in \mathbb{R}$. The architecture builds on the shared principles of [Model Architecture Modifications](11313-model-architecture-modifications.md) by modifying the output head and adding a specific branch for variance estimation.

### Inputs and Outputs
*   **Inputs**:
    *   $\mathbf{x} = (x, y, z)$: 3D coordinates of a point on the ray.
    *   $\mathbf{d} = (d_x, d_y, d_z)$: Viewing direction vector.
*   **Outputs**:
    *   $\sigma$: Volume density.
    *   $\mathbf{c}$: View-dependent color (RGB).
    *   $\nu$: Predicted variance (uncertainty) of the color/density prediction.

### Data Structures and Layout
The network consists of two main components: a Position MLP and a Shading MLP, with a specific modification for variance.

1.  **Positional Encoding**:
    *   Input $\mathbf{x}$ is encoded using $\gamma_{pos}$ with $L=10$ frequency bands.
    *   Input $\mathbf{d}$ is encoded using $\gamma_{dir}$ with $L=4$ frequency bands.
    *   The encoded vectors are concatenated with the original inputs (standard NeRF convention implied by [S1] "follow the configurations in NeRF").
2.  **Position MLP ($\Phi_{pos}$)**:
    *   Input: Encoded $\mathbf{x}$.
    *   Structure: 8 Fully Connected (FC) layers.
    *   Residual Connection: Added to the output of the 4th layer.
    *   Output: A latent feature vector $\mathbf{h}$.
3.  **Density Head**:
    *   Input: Latent feature $\mathbf{h}$.
    *   Structure: Single FC layer.
    *   Output: Volume density $\sigma$.
4.  **Variance Head**:
    *   Input: Latent feature $\mathbf{h}$.
    *   Structure: Single FC layer.
    *   Activation: Softplus.
    *   Output: Variance $\nu$.
5.  **Color Head**:
    *   Input: Concatenation of latent feature $\mathbf{h}$ and encoded viewing direction $\gamma_{dir}(\mathbf{d})$.
    *   Structure: Single FC layer.
    *   Output: RGB color $\mathbf{c}$.

### Positional Encoding Formula
The positional encoding function $\gamma(p)$ for an input $p$ with $L$ bands is defined as [S1]:

$$
\gamma(p) = (\sin(2^0\pi p), \cos(2^0\pi p), \dots, \sin(2^{L-1}\pi p), \cos(2^{L-1}\pi p))
$$

This is applied separately to the 3D coordinates $\mathbf{x}$ and the viewing direction $\mathbf{d}$.

### Parameters
*   $L_{pos} = 10$: Number of frequency bands for 3D coordinates [S1].
*   $L_{dir} = 4$: Number of frequency bands for viewing directions [S1].
*   MLP Depth: 8 layers for the position MLP [S1].
*   Residual Connection: Applied after the 4th layer [S1].

### Pseudocode
The following Python-like pseudocode implements the ActiveNeRF forward pass. It assumes standard vector operations and `torch.nn`-style layer definitions for clarity, but is self-contained in logic.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn
import torch.nn.functional as F

class ActiveNeRFModel(nn.Module):
    def __init__(self):
        super(ActiveNeRFModel, self).__init__()
        
        # Positional Encoding parameters
        self.L_pos = 10
        self.L_dir = 4
        
        # Position MLP: 8 FC layers
        # Layer 0 to 7
        self.pos_mlp_layers = nn.ModuleList([
            nn.Linear(6 * self.L_pos, 256), # Input dim: 2*L_pos (sin/cos)
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256)
        ])
        
        # Density Head: 1 FC layer
        self.density_layer = nn.Linear(256, 1)
        
        # Variance Head: 1 FC layer with Softplus activation
        self.variance_layer = nn.Linear(256, 1)
        
        # Color Head: Concatenates latent feature (256) + encoded direction (2*L_dir)
        # Input dim: 256 + 2 * 4 = 264
        self.color_layer = nn.Linear(256 + 2 * self.L_dir, 3)

    def positional_encoding(self, x, L):
        """
        Applies positional encoding to input x with L bands.
        x shape: (..., 3) for coordinates or (..., 3) for directions.
        Returns shape: (..., 2*L)
        """
        # Reshape to handle batch dimensions
        original_shape = x.shape
        x_flat = x.view(-1, 3)
        
        freq_bands = 2.0 ** torch.arange(0, L, device=x.device).float()
        # freq_bands shape: (L,)
        
        # Expand freq_bands to match x dimensions
        # x_flat shape: (N, 3)
        # freq_bands shape: (L,) -> (1, L)
        # We want to compute sin(2^i * pi * x) for each band i
        
        # Create frequency bands tensor: (1, L)
        freq_bands = freq_bands.view(1, -1)
        
        # Scale input: (N, 3) * (1, L) -> (N, 3, L)
        scaled_x = x_flat.unsqueeze(-1) * (math.pi * freq_bands)
        
        # Compute sin and cos: (N, 3, L)
        sin_vals = torch.sin(scaled_x)
        cos_vals = torch.cos(scaled_x)
        
        # Concatenate along last dim: (N, 3, 2*L)
        encoded = torch.cat([sin_vals, cos_vals], dim=-1)
        
        # Reshape back to original shape with encoded dim
        # New last dim size is 2*L
        new_shape = list(original_shape[:-1]) + [2 * L]
        return encoded.view(new_shape)

    def forward(self, x, d):
        """
        x: (N, 3) 3D coordinates
        d: (N, 3) Viewing directions
        """
        # 1. Positional Encoding
        # NeRF standard: concatenate original input with encoded input
        # However, the note says "encoding function is formulated as... separately applied".
        # Standard NeRF [S1 reference] uses: [x, gamma(x)].
        # Let's follow standard NeRF practice implied by "follow the configurations in NeRF".
        
        x_enc = self.positional_encoding(x, self.L_pos)
        d_enc = self.positional_encoding(d, self.L_dir)
        
        # Concatenate original and encoded
        x_input = torch.cat([x, x_enc], dim=-1) # (N, 3 + 20) = (N, 23)
        # Note: Standard NeRF uses 3 + 2*10 = 23 dims for x.
        
        # 2. Position MLP
        h = x_input
        for i, layer in enumerate(self.pos_mlp_layers):
            h = layer(h)
            if i < 7: # ReLU activation for hidden layers
                h = F.relu(h)
            
            # Residual connection on the 4th layer (index 3)
            # The note says "residual connection on the 4th layer".
            # In a sequence of 8 layers (0-7), the 4th layer is index 3.
            # Typically, residual means h_new = h_old + layer(h_old).
            # Or h_new = layer(h_old) + h_skip.
            # Standard NeRF residual: h = layer(h) + h_skip.
            if i == 3:
                # The input to this layer was h_prev.
                # We need to store h before the layer operation.
                # Since we are iterating, we need to adjust logic.
                # Let's restructure slightly for clarity in pseudocode.
                pass 
        
        # Re-implementing MLP loop with residual logic properly
        h = x_input
        for i, layer in enumerate(self.pos_mlp_layers):
            h_in = h
            h = layer(h)
            if i < 7:
                h = F.relu(h)
            if i == 3:
                # Residual connection: add input of this layer to output
                h = h + h_in
        
        # 3. Density Head
        sigma = self.density_layer(h) # (N, 1)
        
        # 4. Variance Head
        variance = self.variance_layer(h) # (N, 1)
        variance = F.softplus(variance) # Ensure positive variance
        
        # 5. Color Head
        # Concatenate latent feature h with encoded direction
        color_input = torch.cat([h, d_enc], dim=-1) # (N, 256 + 8) = (N, 264)
        color = self.color_layer(color_input) # (N, 3)
        
        return sigma.squeeze(-1), color, variance.squeeze(-1)
```

### Edge Cases and Pitfalls
*   **Variance Positivity**: The variance output must be non-negative. The use of the Softplus activation function in the variance head ensures this [S1].
*   **Residual Connection Placement**: The residual connection is specifically applied after the 4th layer of the 8-layer Position MLP. Misplacing this connection can degrade training stability [S1].
*   **Encoding Dimensions**: The input dimension to the Position MLP is $3 + 2 \times 10 = 23$. The input dimension to the Color Head is $256 + 2 \times 4 = 264$. Incorrect dimensionality in concatenation will cause shape mismatches.

## Operating characteristics
*   **Complexity**: The computational complexity is dominated by the 8-layer MLP for position encoding and the final FC layers. The addition of the variance head adds negligible overhead (one FC layer).
*   **Memory**: The model size is slightly larger than standard NeRF due to the extra output head. The latent feature size remains 256.
*   **Performance**: ActiveNeRF provides uncertainty estimates that can guide active view selection, potentially reducing the number of views needed for high-quality reconstruction compared to standard NeRF. However, training may require careful tuning of the variance loss term to prevent the network from predicting trivially large variances.
*   **Hardware**: The original implementation used a single RTX 2080 Ti GPU [S1].

## Background
ActiveNeRF was introduced in the paper "ActiveNeRF: Learning Where to See with Uncertainty Estimation" [S1]. It builds upon the original NeRF architecture [S1] by integrating uncertainty estimation. The motivation is to enable active learning strategies where the system can select the most informative camera views to capture next, based on the model's uncertainty. This is particularly useful in scenarios where capturing all views is expensive or impractical. The architecture closely follows the original NeRF design but adds a specific branch for variance prediction.

## Sources
[S1] ActiveNeRF: Learning Where to See with Uncertainty Estimation, https://link.springer.com/chapter/10.1007/978-3-031-19827-4_14, fetched 2026-10-28
