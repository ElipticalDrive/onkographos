# Linear Feature Decoder with Learned Color Basis

Also called: Learned color basis (Explicit Model), Linear Combination Decoder, Linear Decoder, explicit k-planes

## Overview
This method replaces the nonlinear Multi-Layer Perceptron (MLP) decoder used in hybrid neural rendering models with a white-box linear decoder that uses spatially-localized features as coefficients for a learned basis to describe view-dependent color and density [S1]. It sets itself apart from its parent [MLPRender_Fea](2414-mlprender-fea.md) by explicitly modeling view dependence through a learned basis rather than a black-box MLP [S1]. This approach enables adaptivity for scenes with variable appearance and avoids the limited expressivity of spherical harmonic decoders [S1].

## How it works
**Inputs:**
- Spatial query point $\mathbf{q}$ [S1].
- View direction $\mathbf{d}$ [S1].
- Spatially-localized feature vector $f(\mathbf{q}) \in \mathbb{R}^M$, where $M$ is the feature dimension [S1].

**Outputs:**
- Predicted color $c(\mathbf{q}, \mathbf{d}) \in [0, 1]^3$ [S1].
- Predicted density $\sigma(\mathbf{q}) \in [0, \infty)$ [S1].

**Data Structures:**
- **Learned Basis Vectors:**
  - $b_\sigma \in \mathbb{R}^M$: A learned basis vector for density, independent of view direction [S1].
  - $b_R(\mathbf{d}), b_G(\mathbf{d}), b_B(\mathbf{d}) \in \mathbb{R}^M$: Learned basis vectors for Red, Green, and Blue channels, dependent on view direction $\mathbf{d}$ [S1].
- **Basis MLP:** A small Multi-Layer Perceptron that maps the view direction $\mathbf{d}$ to the basis vectors $b_R(\mathbf{d}), b_G(\mathbf{d}), b_B(\mathbf{d})$ [S1]. This MLP serves as an adaptive drop-in replacement for spherical harmonic basis functions [S1].

**Parameters:**
- $M$: Feature dimension. Not given in the sources.

**Steps:**
1. **Compute Spatial Features:** Obtain $f(\mathbf{q})$ from the underlying volume representation (e.g., k-planes) [S1]. Note: The linear decoder can only replace an MLP decoder when the underlying planes are multiplied, not added [S1].
2. **Compute Basis Vectors:** Pass the view direction $\mathbf{d}$ through the Basis MLP to obtain $b_R(\mathbf{d}), b_G(\mathbf{d}), b_B(\mathbf{d})$ [S1].
3. **Predict Color:** Compute the dot product of $f(\mathbf{q})$ with each color basis vector and concatenate the results:
   $$c_{raw}(\mathbf{q}, \mathbf{d}) = \bigcup_{i \in \{R, G, B\}} f(\mathbf{q}) \cdot b_i(\mathbf{d})$$
   where $\cdot$ denotes the dot product and $\cup$ denotes concatenation [S1].
4. **Predict Density:** Compute the dot product of $f(\mathbf{q})$ with the density basis vector:
   $$\sigma_{raw}(\mathbf{q}) = f(\mathbf{q}) \cdot b_\sigma$$
   [S1].
5. **Apply Activation Functions:**
   - Apply sigmoid to $c_{raw}(\mathbf{q}, \mathbf{d})$ to force color values into the valid range $[0, 1]$ [S1].
   - Apply exponential to $\sigma_{raw}(\mathbf{q})$ to force density into the valid range $[0, \infty)$, using a truncated gradient during backpropagation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class LinearFeatureDecoder(nn.Module):
    def __init__(self, feature_dim: int, basis_mlp_hidden_dim: int = 64):
        """
        Args:
            feature_dim (int): Dimension M of the spatially-localized features f(q).
            basis_mlp_hidden_dim (int): Hidden dimension for the Basis MLP.
        """
        super().__init__()
        self.feature_dim = feature_dim
        
        # Learned basis vector for density (independent of view direction)
        self.b_sigma = nn.Parameter(torch.randn(feature_dim))
        
        # Basis MLP to generate view-dependent color basis vectors
        # Maps view direction d -> [b_R(d), b_G(d), b_B(d)]
        self.basis_mlp = nn.Sequential(
            nn.Linear(3, basis_mlp_hidden_dim),
            nn.ReLU(),
            nn.Linear(basis_mlp_hidden_dim, 3 * feature_dim) # 3 channels * M features
        )

    def forward(self, features: torch.Tensor, view_dirs: torch.Tensor) -> tuple:
        """
        Args:
            features (torch.Tensor): Spatially-localized features f(q) of shape (..., M).
            view_dirs (torch.Tensor): View directions d of shape (..., 3).
        Returns:
            color (torch.Tensor): Predicted color of shape (..., 3) in [0, 1].
            density (torch.Tensor): Predicted density of shape (...) in [0, inf).
        """
        # 1. Compute view-dependent basis vectors using the Basis MLP
        # basis_mlp_output shape: (..., 3 * M)
        basis_mlp_output = self.basis_mlp(view_dirs)
        
        # Reshape to separate R, G, B channels
        # b_color shape: (..., 3, M)
        b_color = basis_mlp_output.view(*view_dirs.shape[:-1], 3, self.feature_dim)
        
        # 2. Compute raw color via dot product
        # features shape: (..., M)
        # b_color shape: (..., 3, M)
        # dot_product shape: (..., 3)
        raw_color = torch.sum(features.unsqueeze(-2) * b_color, dim=-1)
        
        # 3. Compute raw density via dot product with learned b_sigma
        # b_sigma shape: (M,)
        raw_density = torch.sum(features * self.b_sigma, dim=-1)
        
        # 4. Apply activation functions
        # Sigmoid for color to clamp to [0, 1]
        color = torch.sigmoid(raw_color)
        
        # Exponential for density to clamp to [0, inf)
        # Truncated gradient is applied during backpropagation [G].
        density = torch.exp(raw_density)
        
        return color, density
```

**Edge Cases and Pitfalls:**
- **Plane Factorization:** This linear decoder is only effective when the underlying spatial features are derived from multiplied planes, not added planes. Using it with added planes suggests the MLP in hybrid models is handling spatial structure, which this linear decoder does not [S1].
- **Gradient Truncation:** The exponential function for density requires a truncated gradient to prevent exploding gradients during optimization [S1].

## Subtopics

### Global Appearance Extension
An extension enabling representation of scenes with consistent static geometry but varying lighting or appearance (e.g., time of day) [S1]. It augments the underlying representation (e.g., k-planes) with an $M$-dimensional vector for each training image $t=1\dots T$, which is optimized as a per-image feature vector [S1]. This per-image feature vector is passed as an additional input to either the Basis MLP (in the explicit linear decoder version) or to the MLP color decoder (in the hybrid version), so that it affects color but not geometry [S1].

## Operating characteristics
- **Time Complexity:** Linear in the feature dimension $M$ for the dot product operations. The Basis MLP adds a small constant overhead per view direction [S1].
- **Memory Complexity:** $O(M)$ for storing the learned basis vectors ($b_\sigma$ and the parameters of the Basis MLP) [S1].
- **Performance:** Achieves competitive or state-of-the-art reconstruction fidelity on synthetic and real, static and dynamic scenes [S1].
- **Compression:** Enables high compression rates (e.g., 1000x over a full 4D grid) without relying on MLPs for the final decoding step [S1].
- **Optimization Speed:** Trains orders of magnitude faster than prior implicit models and on par with concurrent hybrid models, despite not using custom CUDA kernels [S1].
- **Strengths:** White-box model, high interpretability, low memory usage, fast optimization, adaptability to variable appearance scenes [S1].
- **Limitations:** Requires multiplied plane factorization for optimal performance [S1].

## Background
The Linear Feature Decoder with Learned Color Basis was introduced in the K-Planes paper by Sara Fridovich-Keil et al. in 2023 [S1]. It was developed to address the limitations of spherical harmonic decoders and black-box MLP decoders in dynamic radiance fields [S1]. The method builds on the k-planes representation, which factorizes 4D volumes into 2D planes [S1]. The approach was inspired by NeX, which proposed using a learned basis for view-dependent color [S1].

## Sources
[S1] K-Planes: Explicit Radiance Fields in Space, Time, and Appearance, https://doi.org/10.1109/cvpr52729.2023.01201, fetched 2026-10-26
